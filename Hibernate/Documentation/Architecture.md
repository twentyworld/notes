## Architecture
---

Hibernate, as an ORM solution, effectively "sits between" the Java application data access layer and the Relational Database, as can be seen in the diagram above. The Java application makes use of the Hibernate APIs to load, store, query, etc its domain data.

As a JPA provider, Hibernate implements the Java Persistence API specifications and the association between JPA interfaces and Hibernate specific implementations can be visualized in the following diagram:

![architecture][1]


#### SessionFactory (org.hibernate.SessionFactory)
A thread-safe (and immutable) representation of the mapping of the application domain model to a database.
Acts as a factory for org.hibernate.Session instances. The EntityManagerFactory is the JPA equivalent of a SessionFactory and basically those two converge into the same SessionFactory implementation.

A SessionFactory is very expensive to create, so, for any given database, the application should have only one associated SessionFactory.

#### Session (org.hibernate.Session)
In JPA nomenclature, the Session is represented by an EntityManager.

Behind the scenes, the Hibernate Session wraps a JDBC java.sql.Connection and acts as a factory for org.hibernate.Transaction instances. It maintains a generally "repeatable read" persistence context (first level cache) of the application domain model.

#### Transaction (org.hibernate.Transaction)
A single-threaded, short-lived object used by the application to demarcate individual physical transaction boundaries. EntityTransaction is the JPA equivalent and both act as an abstraction API to isolate the application from the underlying transaction system in use (JDBC or JTA).


---
## Domain Model
 Sometimes you will also hear the term persistent classes.

### 2.1 Mapping types
 Hibernate understands both the Java and JDBC representations of application data. The ability to read/write this data from/to the database is the function of a Hibernate type.

```Java
create table Contact (
    id integer not null,
    first varchar(255),
    last varchar(255),
    middle varchar(255),
    notes varchar(255),
    starred boolean not null,
    website varchar(255),
    primary key (id)
)


@Entity(name = "Contact")
public static class Contact {
    @Id
    private Integer id;
    private Name name;
    private String notes;
    private URL website;
    private boolean starred;
    //Getters and setters are omitted for brevity
}
@Embeddable
public class Name {
    private String first;
    private String middle;
    private String last;
    // getters and setters omitted
}
```
#### 2.1.1. Value types
A value type is a piece of data that does not define its own lifecycle. It is, in effect, owned by an entity, which defines its lifecycle.
###### Basic types
in mapping the Contact table, all attributes except for name would be basic types. Basic types are discussed in detail in Basic Types
######Embeddable types
the name attribute is an example of an embeddable type, which is discussed in details in Embeddable Types
######Collection types
although not featured in the aforementioned example, collection types are also a distinct category among value types. Collection types are further discussed in Collections

#### 2.1.2. Entity types
Entities, by nature of their unique identifier, exist independently of other objects whereas values do not. Entities are domain model classes which correlate to rows in a database table, using a unique identifier. Because of the requirement for a unique identifier, entities exist independently and define their own lifecycle. The Contact class itself would be an example of an entity.

##@ 2.2 Naming strategies

Part of the mapping of an object model to the relational database is mapping names from the object model to the corresponding database names. Hibernate looks at this as 2 stage process:

- The first stage is determining a proper logical name from the domain model mapping. A logical name can be either explicitly specified by the user (using @Column or @Table e.g.) or it can be implicitly determined by Hibernate through an ImplicitNamingStrategy contract.
- Second is the resolving of this logical name to a physical name which is defined by the PhysicalNamingStrategy contract.

#### 2.2.1. ImplicitNamingStrategy
contract to determine a logical name when the mapping did not provide an explicit name.

There are multiple ways to specify the ImplicitNamingStrategy to use. First, applications can specify the implementation using the hibernate.implicit_naming_strategy [configuration][2].

#### 2.2.2. PhysicalNamingStrategy

There are multiple ways to specify the PhysicalNamingStrategy to use. First, applications can specify the implementation using the hibernate.physical_naming_strategy [configuration][3].

> While the purpose of an ImplicitNamingStrategy is to determine that an attribute named accountNumber maps to a logical column name of accountNumber when not explicitly specified, the purpose of a PhysicalNamingStrategy would be, for example, to say that the physical column name should instead be abbreviated acct_num.

### 2.3 Basic Types
 Hibernate provides a number of built-in basic types, which follow the natural mappings recommended by the JDBC specifications.
 access [hibernate][4] for more information.

#### 2.3.2. The @Basic annotation
a basic type is denoted by the javax.persistence.Basic annotation. Generally speaking, the @Basic annotation can be ignored, as it is assumed by default.

#### 2.3.3. The @Column annotation
For basic type attributes, the implicit naming rule is that the column name is the same as the attribute name. If that implicit naming rule does not meet your requirements, you can explicitly tell Hibernate (and other providers) the column name to use.

#### 2.3.4. BasicTypeRegistry

#### 2.3.5. Explicit BasicTypes
Sometimes you want a particular attribute to be handled differently. Occasionally Hibernate will implicitly pick a BasicType that you do not want (and for some reason you do not want to adjust the BasicTypeRegistry).
```Java
@Entity(name = "Product")
public class Product {

    @Id
    private Integer id;

    private String sku;

    @org.hibernate.annotations.Type( type = "nstring" )
    private String name;

    @org.hibernate.annotations.Type( type = "materialized_nclob" )
    private String description;
}
```
#### 2.3.7. Mapping enums
Hibernate supports the mapping of Java enums as basic value types in a number of different ways.
###### @Enumerated
```Java
@Entity(name = "Phone")
public static class Phone {

    @Id
    private Long id;

    @Column(name = "phone_number")
    private String number;

    @Enumerated(EnumType.ORDINAL)
    @Column(name = "phone_type")
    private PhoneType type;

    //Getters and setters are omitted for brevity

}
```
###### AttributeConverter

#### 2.3.8 Mapping LOBs
Mapping LOBs (database Large Objects) come in 2 forms, those using the JDBC locator types and those materializing the LOB data.
```Java
CREATE TABLE Product (
  id INTEGER NOT NULL
  image clob
  name VARCHAR(255)
  PRIMARY KEY ( id )
)

@Entity(name = "Product")
public static class Product {
    @Id
    private Integer id;
    private String name;
    @Lob
    private Clob warranty;
    //Getters and setters are omitted for brevity
}
```

#### 2.3.15. Mapping Date/Time Values
```Java
@Entity(name = "DateEvent")
public static class DateEvent {

    @Id
    @GeneratedValue
    private Long id;

    @Column(name = "`timestamp`")
    @Temporal(TemporalType.DATE)
    private Date timestamp;

    //Getters and setters are omitted for brevity

}
```
###### @CreationTimestamp annotation
The @CreationTimestamp annotation instructs Hibernate to set the annotated entity attribute with the current timestamp value of the JVM when the entity is being persisted.

###### @UpdateTimestamp annotation
The @UpdateTimestamp annotation instructs Hibernate to set the annotated entity attribute with the current timestamp value of the JVM when the entity is being persisted.
When updating the entity, Hibernate is going to modify the column with the current JVM timestamp

#### 2.3.20. @Formula
Sometimes, you want the Database to do some computation for you rather than in the JVM, you might also create some kind of virtual column.
```Java
@Entity(name = "Account")
public static class Account {

    @Id
    private Long id;

    private Double credit;

    private Double rate;

    @Formula(value = "credit * rate")
    private Double interest;

    //Getters and setters omitted for brevity

}
```
#### 2.3.21. @Where
Sometimes, you want to filter out entities or collections using a custom SQL criteria. This can be achieved using the @Where annotation, which can be applied to entities and collections.
```Java
public enum AccountType {
    DEBIT,
    CREDIT
}

@Entity(name = "Client")
public static class Client {

    @Id
    private Long id;

    private String name;

    @Where( clause = "account_type = 'DEBIT'")
    @OneToMany(mappedBy = "client")
    private List<Account> debitAccounts = new ArrayList<>( );

    @Where( clause = "account_type = 'CREDIT'")
    @OneToMany(mappedBy = "client")
    private List<Account> creditAccounts = new ArrayList<>( );

    //Getters and setters omitted for brevity

}

@Entity(name = "Account")
@Where( clause = "active = true" )
public static class Account {

    @Id
    private Long id;

    @ManyToOne
    private Client client;

    @Column(name = "account_type")
    @Enumerated(EnumType.STRING)
    private AccountType type;

    private Double amount;

    private Double rate;

    private boolean active;

    //Getters and setters omitted for brevity

}
```
#### 2.3.23. @Filter
The @Filter annotation is another way to filter out entities or collections using a custom SQL criteria, for both entities and collections. Unlike the @Where annotation, @Filter allows you to parameterize the filter clause at runtime.




### 2.4 Embeddable types
```Java
@Entity(name = "Book")
public static class Book {
    @Id
    @GeneratedValue
    private Long id;
    private String title;
    private String author;
    private Publisher publisher;
    //Getters and setters are omitted for brevity
}

@Embeddable
public static class Publisher {
    @Column(name = "publisher_name")
    private String name;
    @Column(name = "publisher_country")
    private String country;
    public Publisher(String name, String country) {
        this.name = name;
        this.country = country;
    }
    private Publisher() {}
    //Getters and setters are omitted for brevity
}


create table Book (
    id bigint not null,
    author varchar(255),
    publisher_country varchar(255),
    publisher_name varchar(255),
    title varchar(255),
    primary key (id)
)
```
> JPA defines two terms for working with an embeddable type: @Embeddable and @Embedded.
> - @Embeddable is used to describe the mapping type itself (e.g. Publisher).
> - @Embedded is for referencing a given embeddable type (e.g. book#publisher).















































[1]:http://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/images/architecture/JPA_Hibernate.svg
[2]:http://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/Hibernate_User_Guide.html#ImplicitNamingStrategy
[3]:http://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/Hibernate_User_Guide.html#ImplicitNamingStrategy
[4]:http://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/Hibernate_User_Guide.html#basic-provided
