# Sirportly Ruby API

This library allows you to interact with your Sirportly data using PHP.

## Setting up a Sirportly Client

```php
$sirportly = new Sirportly('the-token','the-secret');
```

## Creating a ticket

You can create tickets within your Sirportly system with a few commands. It's important to note that
creating a new ticket is a two step process - firstly, you need to create a `Ticket` record and then
you need to post your initial update using `post_update` on your newly created ticket.

```php
# Create the skeleton ticket
$properties = array(
    'brand' => 'Sirportly', 
    'department' => 'Sales Enquiries',
    'status' => 'New',
    'priority' => 'Normal',
    'subject' => 'A new sales enquiry',
    'name' => 'My New Customer',
    'email' => 'customer@atechmedia.com',
    );
    
  $ticket = $sirportly->create_ticket($properties);

# Now add the first update to this ticket
$update = $sirportly->post_update(array('ticket' => $ticket['reference'], 'message' => 'I would like some more info about your product', 'customer' => $ticket['customer']['id'] ));
```

If an error occurs, you will receive an array of errors. There are many
other properties which can be passed to the `create_ticket` method which are not documented here. Take
a look at the [API documentation](http://www.sirportly.com/docs/api-specification/tickets/submitting-a-new-ticket)
for more information about the options available.

## Accessing Tickets

```php
sirportly.tickets                   
sirportly.ticket('AB-123123')       
```

You can also access tickets through filter objects.

```ruby
filter = sirportly.filters.first    #=> A Sirportly::Filter object
filter.tickets                      #=> A Sirportly::DataSet of objects (paginated)
filter.tickets(:page => 2)          #=> The second page of tickets
filter.tickets(:user => 'adam')     #=> The tickets as if being accessed by 'adam'
```

## Changing ticket properties

If you wish to change properties of a ticket, you can use the `update` method. This method behaves
exactly the same as the corresponding API method and further details can be found in the 
[documentation](https://atech.sirportly.com/knowledge/4/api-specification/tickets/changing-ticket-properties). 
You can pass strings, IDs or `Sirportly::DataObject` objects as values. 

```ruby
ticket = sirportly.ticket('AB-123123')

# Change a ticket status
ticket.update(:status => "waiting for staff")

# Change a ticket priority
ticket.update(:priority => "low")

# Change multiple attributes
ticket.update(:team => "1st line support", :user => "dave")
```

Once an update has been carried out, the original ticket object will be updated to include the new properties.

## Posting updates to tickets

Posting updates to tickets is a simple affair and the `post_update` method on a `Sirportly::Ticket`
will accept the same parameters as defined in the [documentation](http://www.sirportly.com/docs/api-specification/tickets/posting-an-update).

As you will see from the examples below, you can pass a `Sirportly::User` instance to `user` and a 
`Sirportly::Customer` instance to `customer` although strings are perfectly acceptable too.

The `post_update` method will return a `Sirportly::TicketUpdate` instance and the new update will
be added to the `updates` array on the original ticket.

```ruby
ticket = sirportly.ticket('AB-123123')

# To post a system message without a user
ticket.post_update(:message => "My Example Message")

# To post an update as the ticket customer
ticket.post_update(:message => "My Example Message", :customer => ticket.customer)

# To post an update as a user
user = sirportly.user('adam')
ticket.post_update(:message => "My Example Message", :user => user)

# To post an update and e-mail it to the customer
ticket.post_update(:message => "My Example", :user => 'adam', :outbound_address => 'support@yourdomain.com')

# To post a private update as a user
ticket.post_update(:message => "Private Msg", :private => true, :user => 'charlie')
```

## Executing Macros

If you wish to execute one of your macros on a ticket, you can use the `run_macro` method
which accepts the ID or name of the macro you wish to execute. If executed successfully,
it will return true and the original ticket properties will be updated. If it fails, an
exception will be raised or the method will return false.

```ruby
ticket = sirportly.ticket('AB-123123')
ticket.run_macro('Mark as waiting for staff')
````

## Adding follow ups

Adding to follow ups to tickets can be achieved by executing the `add_follow_up` method on a 
`Sirportly::Ticket` instance.

```ruby
ticket = sirportly.ticket('AB-123123')
ticket.add_follow_up(:actor => 'adam', :status => 'resolved', :run_at => '2 days from now') #=> true
```

The `run_at` attribute should be a timestamp as outlined on our
[date/time formatting page](http://www.sirportly.com/docs/api-specification/date-time-formatting) in 
the API documentation.

## Creating a user

You can create users (staff members) via the API.

```ruby
user_properties = {
  :first_name    => 'John',
  :last_name     => 'Particle',
  :email_address => 'john@testcompany.com',
  :admin_access  => true
}

user = sirportly.create_user(user_properties)  #=> A Sirportly::User instance
```

There are other attributes available, which can be viewed on the [API docs](http://www.sirportly.com/docs/api-specification/users/create-new-user).

You do not need to create individual customers. These are created automatically on ticket and ticket update creation.

## Accessing Static Data Objects

The Sirportly API provides access to all the data objects stored in your Sirportly database.
At the current time, these cannot be edited through the API. 

```ruby
sirportly.statuses                  #=> Set of all statuses as Sirportly::Status objects
sirportly.priorities.first          #=> A Sirportly::Priority object for the first record

sirportly.brands.first.departments  #=> Array of Sirportly::Department objects

sirportly.user('adam')              #=> Returns a Sirportly::User object
sirportly.customer('Dave Smith')    #=> Returns a Sirportly::Customer object
```

You can access the following objects using this method: brands, departments, escalation_paths,
filters, priorities, slas, statuses, teams and users.

## Pagination

Some results from the API are paginated as outlined below. By default, it will always 
return the first page.

```ruby
users = sirportly.users(:page => 1)

users.each do |user|
  user.is_a?(Sirportly::User)     #=> true
  user.full_name                  #=> "Adam Cooke"
  user.teams                      #=> Array of Sirportly::Team objects
  user.teams.first.name           #=> "First Line Support"
end

users.page              #=> 1
users.total_records     #=> 35
users.pages             #=> 2
users.offset            #=> 0
```

If a result set is not paginated, the methods outlined above will be nil. Pagination will only occur 
at the top level of results and does not happen on arrays within objects.

## Executing SPQL queries

Sirportly includes a powerful query language called SPQL (SirPortly Query Language) which allows you
to query your ticket data through the API. This is primarily used to generate reports however can also
be used to return data for your own purposes.

```ruby
query = sirportly.spql('SELECT COUNT, brand.name FROM tickets GROUP BY brand.name')
query.fields          #=> ["COUNT", "brand.name"]
query.results         #=> [[123, "Appli"], [456, "aTech Media"], [789, "aTech Telecoms"], [123, "Sirportly"]]
query.query           #=> "SELECT COUNT, brand.name FROM tickets GROUP BY brand.name"
query.class.to_s      #=> Sirportly::SPQLQuery
```

If you execute a query which is invalid, a `Sirportly::Errors::ValidationError` will be raised with some
information about the error.

```php
query = sirportly.spql('SELECT COUNT FROM non_existent_table')
Sirportly::Errors::ValidationError: ["Invalid FROM table specified"]
```
