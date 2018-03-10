Spring Data Commons

The goal of Spring Data repository abstraction is to significantly reduce the amount of boilerplate code required to implement data access layers for various persistence stores.

---

### 1. Working with Spring Data Repositories
#### 1.1 Core concepts
The central interface in Spring Data repository abstraction is Repository.

We also provide persistence technology-specific abstractions like e.g. JpaRepository or MongoRepository. Those interfaces extend CrudRepository and expose the capabilities of the underlying persistence technology in addition to the rather generic persistence technology-agnostic interfaces like e.g. CrudRepository.

#### 1.2 Query methods.
Standard CRUD functionality repositories usually have queries on the underlying datastore. With Spring Data, declaring those queries becomes a four-step process:


1. Declare an interface extending Repository or one of its subinterfaces and type it to the domain class and ID type that it will handle.
```
interface PersonRepository extends Repository<Person, Long> { … }
```
2. Declare query methods on the interface.
```
interface PersonRepository extends Repository<Person, Long> {
  List<Person> findByLastname(String lastname);
}
```

3. Set up Spring to create proxy instances for those interfaces. Either via JavaConfig:
```
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;

@EnableJpaRepositories
class Config {}
```
4. Get the repository instance injected and use it.
```
class SomeClient {

  private final PersonRepository repository;

  SomeClient(PersonRepository repository) {
    this.repository = repository;
  }

  void doSomething() {
    List<Person> persons = repository.findByLastname("Matthews");
  }
}
```

#### 1.3 Defining repository interfaces
As a first step you define a domain class-specific repository interface. The interface must extend Repository and be typed to the domain class and an ID type. If you want to expose CRUD methods for that domain type, extend CrudRepository instead of Repository.
> 在repsoitory中并没有提供任何的方法，但是在CrudRepository中提供了一些简单的CURD查询。
```
public interface Repository<T, ID extends Serializable> {
}


public interface CrudRepository<T, ID extends Serializable> extends Repository<T, ID> {
	<S extends T> S save(S entity);
	<S extends T> Iterable<S> save(Iterable<S> entities);
	T findOne(ID id);
	boolean exists(ID id);
	Iterable<T> findAll();
	Iterable<T> findAll(Iterable<ID> ids);
	long count();
	void delete(ID id);
	void delete(T entity);
	void delete(Iterable<? extends T> entities);
	void deleteAll();
}
```
##### 1.3.1 Fine-tuning repository definition
your repository interface will extend Repository, CrudRepository or PagingAndSortingRepository. Alternatively, if you do not want to extend Spring Data interfaces, you can also annotate your repository interface with @RepositoryDefinition.

> This allows you to define your own abstractions on top of the provided Spring Data Repositories functionality.

```
@NoRepositoryBean
interface MyBaseRepository<T, ID extends Serializable> extends Repository<T, ID> {
  Optional<T> findById(ID id);
  <S extends T> S save(S entity);
}

interface UserRepository extends MyBaseRepository<User, Long> {
  User findByEmailAddress(EmailAddress emailAddress);
}
```
> Note, that the intermediate repository interface is annotated with @NoRepositoryBean. Make sure you add that annotation to all repository interfaces that Spring Data should not create instances for at runtime.




#### 1.4 Defining query methods

##### Limiting query results
The results of query methods can be limited via the keywords first or top, which can be used interchangeably. An optional numeric value can be appended to top/first to specify the maximum result size to be returned. If the number is left out, a result size of 1 is assumed.

##### Async query results

Repository queries can be executed asynchronously using Spring’s asynchronous method execution capability. This means the method will return immediately upon invocation and the actual query execution will occur in a task that has been submitted to a Spring TaskExecutor.
```
>	Use java.util.concurrent.Future as return type.
@Async
Future<User> findByFirstname(String firstname);
>Use a Java 8 java.util.concurrent.CompletableFuture as return type.
@Async
CompletableFuture<User> findOneByFirstname(String firstname);
>Use a org.springframework.util.concurrent.ListenableFuture as return type.
@Async
ListenableFuture<User> findOneByLastname(String lastname);
```

#### 1.5 Publishing events from aggregate roots
Entities managed by repositories are aggregate roots. In a Domain-Driven Design application, these aggregate roots usually publish domain events. Spring Data provides an annotation @DomainEvents you can use on a method of your aggregate root to make that publication as easy as possible.
