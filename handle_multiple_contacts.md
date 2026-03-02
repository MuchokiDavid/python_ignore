You're right to identify that redundancy - having duplicate contact data across multiple models is a common database design issue. Here's a comprehensive solution to normalize your contact data into a single model:

## Recommended Database Schema

### 1. Single `contacts` Table
```python
# models.py
from django.db import models
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType

class Contact(models.Model):
    # Core contact information
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    email = models.EmailField(unique=True, blank=True, null=True)
    phone = models.CharField(max_length=20, blank=True)
    mobile = models.CharField(max_length=20, blank=True)
    
    # Address information
    address_line1 = models.CharField(max_length=255, blank=True)
    address_line2 = models.CharField(max_length=255, blank=True)
    city = models.CharField(max_length=100, blank=True)
    state = models.CharField(max_length=100, blank=True)
    postal_code = models.CharField(max_length=20, blank=True)
    country = models.CharField(max_length=100, blank=True)
    
    # Metadata
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_active = models.BooleanField(default=True)
    
    # For associating with different entity types
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE, null=True, blank=True)
    object_id = models.PositiveIntegerField(null=True, blank=True)
    associated_entity = GenericForeignKey('content_type', 'object_id')
    
    # Type designation (can be multiple through a junction table)
    contact_types = models.ManyToManyField('ContactType', through='ContactTypeAssignment')
    
    def __str__(self):
        return f"{self.first_name} {self.last_name}"
    
    class Meta:
        indexes = [
            models.Index(fields=['content_type', 'object_id']),
            models.Index(fields=['email']),
        ]

class ContactType(models.Model):
    name = models.CharField(max_length=50, unique=True)  # 'lead', 'client', 'debtor', etc.
    description = models.TextField(blank=True)
    
    def __str__(self):
        return self.name

class ContactTypeAssignment(models.Model):
    contact = models.ForeignKey(Contact, on_delete=models.CASCADE)
    contact_type = models.ForeignKey(ContactType, on_delete=models.CASCADE)
    assigned_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        unique_together = ['contact', 'contact_type']
```

### 2. Entity Models (Lead, Client, Debtor)
```python
class Lead(models.Model):
    company_name = models.CharField(max_length=200)
    industry = models.CharField(max_length=100, blank=True)
    lead_source = models.CharField(max_length=100, blank=True)
    status = models.CharField(max_length=50, default='new')
    
    # Primary contact (optional - can use generic relation)
    primary_contact = models.ForeignKey(
        Contact, 
        on_delete=models.SET_NULL, 
        null=True, 
        related_name='primary_for_leads'
    )
    
    # All contacts associated with this lead
    contacts = GenericRelation(
        Contact, 
        content_type_field='content_type',
        object_id_field='object_id',
        related_query_name='lead'
    )
    
    created_at = models.DateTimeField(auto_now_add=True)
    converted_to_client = models.ForeignKey(
        'Client', 
        on_delete=models.SET_NULL, 
        null=True, 
        blank=True
    )
    
    def convert_to_client(self, user=None):
        """Convert lead to client without duplicating contacts"""
        client = Client.objects.create(
            company_name=self.company_name,
            industry=self.industry,
            lead_source=self.lead_source,
            converted_from=self,
            converted_by=user,
            converted_at=timezone.now()
        )
        
        # Reassign all contacts to the client
        for contact in self.contacts.all():
            # Add client type to contact
            client_type = ContactType.objects.get(name='client')
            ContactTypeAssignment.objects.get_or_create(
                contact=contact,
                contact_type=client_type
            )
        
        self.converted_to_client = client
        self.save()
        return client

class Client(models.Model):
    company_name = models.CharField(max_length=200)
    industry = models.CharField(max_length=100, blank=True)
    lead_source = models.CharField(max_length=100, blank=True)
    
    # Relationship to lead
    converted_from = models.OneToOneField(
        Lead, 
        on_delete=models.SET_NULL, 
        null=True, 
        blank=True,
        related_name='client_conversion'
    )
    converted_at = models.DateTimeField(null=True, blank=True)
    converted_by = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.SET_NULL,
        null=True,
        blank=True
    )
    
    # Primary contact
    primary_contact = models.ForeignKey(
        Contact,
        on_delete=models.SET_NULL,
        null=True,
        related_name='primary_for_clients'
    )
    
    # All contacts for this client
    contacts = GenericRelation(
        Contact,
        content_type_field='content_type',
        object_id_field='object_id',
        related_query_name='client'
    )
    
    contract_start_date = models.DateField(null=True, blank=True)
    contract_end_date = models.DateField(null=True, blank=True)
    is_active = models.BooleanField(default=True)

class Debtor(models.Model):
    company_name = models.CharField(max_length=200)
    debt_amount = models.DecimalField(max_digits=10, decimal_places=2)
    due_date = models.DateField()
    payment_terms = models.CharField(max_length=100, blank=True)
    
    # Relationship to client
    client = models.ForeignKey(
        Client, 
        on_delete=models.CASCADE,
        related_name='debtors'
    )
    
    # Primary contact
    primary_contact = models.ForeignKey(
        Contact,
        on_delete=models.SET_NULL,
        null=True,
        related_name='primary_for_debtors'
    )
    
    # All contacts for this debtor
    contacts = GenericRelation(
        Contact,
        content_type_field='content_type',
        object_id_field='object_id',
        related_query_name='debtor'
    )
    
    status = models.CharField(max_length=50, default='outstanding')  # outstanding, paid, overdue
```

### 3. Helper Methods and Migration Script

```python
# utils.py
from django.contrib.contenttypes.models import ContentType

def migrate_existing_data():
    """One-time migration script to consolidate existing data"""
    
    # Migrate Lead contacts
    for lead in Lead.objects.all():
        lead_ct = ContentType.objects.get_for_model(lead)
        
        # Check if contact already exists for this lead
        existing_contacts = Contact.objects.filter(
            content_type=lead_ct,
            object_id=lead.id
        )
        
        if not existing_contacts.exists() and hasattr(lead, 'contact'):
            # Create contact from old Lead.contact data
            contact = Contact.objects.create(
                first_name=lead.contact.first_name,
                last_name=lead.contact.last_name,
                email=lead.contact.email,
                phone=lead.contact.phone,
                content_type=lead_ct,
                object_id=lead.id
            )
            
            # Assign lead type
            lead_type = ContactType.objects.get_or_create(name='lead')[0]
            ContactTypeAssignment.objects.create(
                contact=contact,
                contact_type=lead_type
            )
    
    # Similar migration for ClientContact and DebtorContact...

class ContactManager:
    @staticmethod
    def get_contacts_for_entity(entity, contact_type=None):
        """Get all contacts for an entity, optionally filtered by type"""
        ct = ContentType.objects.get_for_model(entity)
        contacts = Contact.objects.filter(content_type=ct, object_id=entity.id)
        
        if contact_type:
            contacts = contacts.filter(
                contacttypeassignment__contact_type__name=contact_type
            )
        
        return contacts
    
    @staticmethod
    def add_contact_to_entity(entity, contact_data, contact_types=None):
        """Add a contact to an entity with optional type assignments"""
        ct = ContentType.objects.get_for_model(entity)
        
        contact = Contact.objects.create(
            **contact_data,
            content_type=ct,
            object_id=entity.id
        )
        
        if contact_types:
            for type_name in contact_types:
                contact_type = ContactType.objects.get_or_create(name=type_name)[0]
                ContactTypeAssignment.objects.create(
                    contact=contact,
                    contact_type=contact_type
                )
        
        return contact
    
    @staticmethod
    def transfer_contact(contact, from_entity, to_entity):
        """Transfer a contact from one entity to another"""
        ct = ContentType.objects.get_for_model(to_entity)
        contact.content_type = ct
        contact.object_id = to_entity.id
        contact.save()
```

### 4. Usage Examples

```python
# Creating a new lead with contacts
lead = Lead.objects.create(company_name="Acme Corp")

# Add contacts
contact = ContactManager.add_contact_to_entity(
    lead,
    {
        'first_name': 'John',
        'last_name': 'Doe',
        'email': 'john@acme.com',
        'phone': '123-456-7890'
    },
    contact_types=['lead', 'primary']
)

# Converting lead to client (no duplication!)
client = lead.convert_to_client()

# The same contact now appears for both lead and client
lead_contacts = ContactManager.get_contacts_for_entity(lead)
client_contacts = ContactManager.get_contacts_for_entity(client)
# Both return the same contact instance!

# Adding a contact to client that was originally a lead
ContactManager.add_contact_to_entity(
    client,
    {
        'first_name': 'Jane',
        'last_name': 'Smith',
        'email': 'jane@acme.com'
    },
    contact_types=['client', 'billing']
)

# Finding all contacts with multiple roles
all_client_contacts = Contact.objects.filter(
    contacttypeassignment__contact_type__name='client'
)
```

## Benefits of This Approach

1. **No Data Duplication**: Contacts are stored once and referenced by multiple entities
2. **Flexible Relationships**: A contact can be associated with leads, clients, and debtors simultaneously
3. **Type System**: Contacts can have multiple roles (e.g., a person could be both a client contact and a debtor)
4. **Easy Migration**: You can gradually migrate existing data without breaking functionality
5. **Better Querying**: Easier to find all contacts across the system or specific subsets

## Migration Strategy

1. Create the new models first
2. Write a data migration to move existing contacts to the new structure
3. Update your code to use the new `Contact` model and `ContactManager`
4. Gradually deprecate and remove the old contact models
5. Add indexes and optimize queries as needed

This design maintains data integrity, eliminates redundancy, and provides flexibility for future requirements like a unified contact search across all entity types.
