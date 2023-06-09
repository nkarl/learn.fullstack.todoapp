## I. Start A New Project
---

Start a new project:

- via the *SpringInitializr* at https://start.spring.io/, or
- using the Spring CLI:
```sh
# for pwsh; for other shells use "\" instead of "`"
spring init --dependencies='web,jpa,security,devtools' `
			--build=gradle `
			--type=gradle-project-kotlin `
			--group-id='learn.fullstack' `
			todoApp
```


## II. Connecting Spring Web with Hibernate
---

```mermaid
graph LR;
	HTML --> UserTDO
	JSON --> UserTDO
	subgraph Presentation Layers
		subgraph Data Transfer Object
			UserTDO[-name\n-roles]
		end
		subgraph User Details
			User[-name\n-password]
		end
		subgraph User Role
			Role[-name]
		end
	UserTDO-->Mapper
	Mapper --> User
	Mapper --> Role
	end
	POJO[Flat Data Objects]
	User --- POJO
	Role --- POJO
```


The steps for connecting Spring with Hibernate are outlined below.

### 1. Configure the Database

1. ensure JDBC driver
   - _which should be included when initializing with JPA_
2. create database schemas and tables

### 2. Configure Spring

1. create Spring config file to define:
   - Spring Beans, and their
   - dependencies
2. configure base beans for:
   - data source: `DataSource`
   - session management: `SessionFactory`
   - transaction management

### 3. Configure Hibernate

This step encompassing setting up the details for database connection, dialect and other properties.

#### JBDC vs. JPA

JPA provides another abstraction layer on top of JDBC. With JPA, the need to write SQL queries like with JDBC is eliminated. Database queries are mapped directly to Java objects and are managed via an entity manager as shown below:

```java
// JBDC sample
@Repository
public class CourseJbdcRepository {

    @Autowired
    private JdbcTemplate springJdbcTemplate;

    private static String INSERT_QUERY = """
                INSERT INTO course(id, name, author)
                VALUES(?, ?, ?);
            """;

    public void insert(Course course) {
        springJdbcTemplate.update(INSERT_QUERY,
                course.getId(), course.getName(), course.getAuthor());
    }
}
```

```java
// JPA sample
@Repository
@Transactional
public class UserJpaRepository {
	@PersistenceContext
	private EntityManager entityManager;

	public void insert(User user) {
		entity.Manager.merge(user);
	}
}
```


In short, the differences between JBDC and JPA are summarized in the following table:

| JBDC                                                                                | JPA                                                                                                                    |
| ----------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| We write SQL commands to a relational database.                                     | We construct database-driven Java programs using OOP semantics.                                                        |
| We *need to write out the full SQL query* for associating database tables.          | We *only need annotations* to create one-to-one, one-to-many, many-to-one, and many-to-many associations.              |
|                                                                                     | JPA handles all the time-consuming, error-prone coding required to convert between OOP Java code and backend database. |
| Is database-dependent. *Different scripts must be written for different databases*. | Is database-agnostic. *Same Java code can be used in a variety of databases with few/no modifications*.                       | 

| JPA                                        | Hibernate                                    |
| ------------------------------------------ | -------------------------------------------- |
| Is an *interface*.                         | Is the most popular *implementation of JPA*. |
| How do you define entities? `@Entity`      |                                              |
| How do you map attributes? `@Column`       |                                              |
| Who manages the entities? entity managers. |                                              |
| *Defines how* you can map the objects.       | *Implements* the mapping.                      | 

### 4. Define Entity Classes

1. create Entity Classes (EC) that represents tables
2. decorate EC with Hibernate annotations
   - such as `@Entity`, `@Table`, `@Column`, etc.
   - to map EC with corresponding tables and columns.

Sample code in Kotlin & Java:
```kotlin
import javax.persistence.*

@Entity
@Table(name = "users")
data class User(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(name = "username")
    val username: String,

    @Column(name = "email")
    val email: String
)
```

```java
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Entity
@Table
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
	@Column(nullable = false)
    private String name;
	@Column(nullable = false)
    private String email;

    // Constructors, getters, and setters
    public User() { }

    public User(String name, String email) {
        this.name = name;
        this.email = email;
    }

    public Long   getId  ()            { return id;[]() }
    public void   setId  (Long id)     { this.id = id; }
    public String getName()            { return name; }
    public void   setName(String name) { this.name = name; }

```

> [!important]
> It's important to note that the use of `Optional` is a design decision and should be used judiciously. It is generally recommended to avoid wrapping non-null fields with `Optional`, as it adds unnecessary complexity. Only use `Optional` when dealing with genuinely optional values that can be absent or null.

### 5. Implement Data Access Objects (DAO)

1. create DAO classes to encapsulate database operations and interact with Hibernate to perform CRUD operations
2. use Hibernates to execute database queries and manage transactions

The interface for `UserDAO` in Kotlin & Java:

```kotlin
interface UserDao {
    fun findById(id: Long): User?
    fun findAll(): List<User>
    fun save(user: User)
    fun update(user: User)
    fun delete(user: User)
}
```

```java
import java.util.List;

public interface UserDao {
    User findById(long id);
    List<User> findAll();
    void save(User user);
    void update(User user);
    void delete(User user);
}
```

Sample code for DAO classes in Kotlin & Java:

```kotlin
import org.hibernate.SessionFactory
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.stereotype.Repository

// assumption: `User` exists.
@Repository
class UserDaoImpl @Autowired constructor(private val sessionFactory: SessionFactory) : UserDao {

    override fun findById(id: Long): User? {
        val session = sessionFactory.currentSession
        return session.get(User::class.java, id)
    }

    override fun findAll(): List<User> {
        val session = sessionFactory.currentSession
        val query = session.createQuery("FROM User", User::class.java)
        return query.resultList
    }

    override fun save(user: User) {
        val session = sessionFactory.currentSession
        session.save(user)
    }

    override fun update(user: User) {
        val session = sessionFactory.currentSession
        session.update(user)
    }

    override fun delete(user: User) {
        val session = sessionFactory.currentSession
        session.delete(user)
    }
}
```

```java
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public class UserDaoImpl implements UserDao {

    private final SessionFactory sessionFactory;

    @Autowired
    public UserDaoImpl(SessionFactory sessionFactory) {
        this.sessionFactory = sessionFactory;
    }

    @Override
    public User findById(long id) {
        Session session = sessionFactory.getCurrentSession();
        return session.get(User.class, id);
    }

    @Override
    public List<User> findAll() {
        Session session = sessionFactory.getCurrentSession();
        return session.createQuery("FROM User", User.class).list();
    }

    @Override
    public void save(User user) {
        Session session = sessionFactory.getCurrentSession();
        session.save(user);
    }

    @Override
    public void update(User user) {
        Session session = sessionFactory.getCurrentSession();
        session.update(user);
    }

    @Override
    public void delete(User user) {
        Session session = sessionFactory.getCurrentSession();
        session.delete(user);
    }
}
```

### 6. Enable Transaction Management

- _In general, database operations should be grouped into a transaction if they are in a logical unit of work_. There are few guidelines to consider:
  - **Atomicity:** operations that should be executed atomically, i.e. either they all succeed or all fail. This happens when we need to update multiple related tables. All updated are applied or none at all for data integrity.
  - **Data Consistency:** this is in the case when there are dependencies between records. For example, transferring funds between bank account must ensure the modifications made to the sender and the recipient accounts are in the same transaction.
  - **Entity Relationships (ER):** grouping related entities together ensures that the relationships are properly maintained.
  - **Performance:** there is an overhead to transactions: resource locking, logging, and transaction bookkeeping.
- _Transactions should be kept as short as possible for efficiency in terms of resource locking and concurrency optimization_.


## III. Conclusion
---

Above is a detailed model of connecting Spring and Hibernate together.

There are few things I can consider next:

- use a mock dataset for banking data to practice building a backend:
	 - get the mock dataset from:
		 - https://www.mockaroo.com/
		 - I will need to design the schemas for user
		 - TASK: research typical schemas for banking data
	 - build APIs with Spring and Hibernate
- build a simple front end with React
	- should have a very bareboned table containing records
- extend the system to include more services
	- authentication (via JWT)
	- etc.
 