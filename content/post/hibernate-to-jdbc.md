---
authors:
- heatherf
categories:
- Spring
- Java
- Hibernate
- JDBC
- Java Database Connectivity
- Repositories
date: 2017-06-01T08:53:03-07:00
draft: true
short: An exploration in replacing Hibernate with JDBC in Spring.
title: Hibernate to JDBC
---

Our team at Pivotal creates and maintains many Spring apps. We have been using [Hibernate](http://hibernate.org/) for data access and persistence. Because of being bitten by some of the subtleties of Hibernate, we decided to try to replace Hibernate with the lower-level JDBC Repository in one of our apps.

Here's why and how.

## Problems with Hibernate

Because we practice test-driven development, many of our problems stemmed from Hibernate in the testing environment. When using Hibernate to persist data, it will not save the data to the database without `entityManager.persist` and `entityManager.flush`.

Each of our tests are transactions so that we can roll back the testing database if a test fails. This got tricky when we had Hibernate transactions wrapped inside of our test transactions and the Hibernate transaction would fail. When debugging these failed tests, the lines of code we traced them to would be deep inside the Hibernate/Spring code instead of in our code. We lost a lot of time to debugging these issues.

This was manifested when our test data wouldn't hit database constraints until the test completed. This gave us problems when trying to test those database constraints. For example, if a table column had a uniqueness constraint on it and in our test we violated that constraint, the test would fail in the Spring code while the transaction was finishing up instead of earlier where we had made the erroneous object.

When we were considering removing Hibernate, we did look into using an already-made JDBC repository library such as this [Spring Data JDBC generic DAO implementation](https://github.com/jirutka/spring-data-jdbc-repository), however it was not a good fit for us because we have many-to-many relationships. If we had simple CRUD uses, it would have been a better fit, but we also might not have needed to move away from Hibernate if that were the case.


## Sample Hibernate Model and Repository

### Garden Model
```
@Data
@Builder
@EqualsAndHashCode
@AllArgsConstructor
@NoArgsConstructor
@Entity(name = "gardens")
public class Garden {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    public void addFlowers(Flower... flowers) {
      if (this.flowers == null) {
        this.flowers = new HashSet<>();
      }

      for (Flower flower : flowers) {
        flower.setGarden(this);
        this.flowers.add(flower);
      }
    }
}

```

### Flower Model
```
@Data
@EqualsAndHashCode(exclude = {"garden"})
@Entity(name = "flower")
public class Flower {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @Column(name = "created_at")
    private ZonedDateTime createdAt;

    @ManyToOne
    private Garden garden;

    @OneToMany(cascade = CascadeType.ALL, mappedBy = "flower", orphanRemoval = true)
}
```

### Garden Repository
```
@Component
public class GardenRepository {
    private final GardenDatabaseRepository gardenDatabaseRepository;

    @Autowired
    GardenRepository(GardenDatabaseRepository gardenDatabaseRepository) {
        this.gardenDatabaseRepository = gardenDatabaseRepository;
    }

    public Iterable<Garden> findAll() {
        return gardenDatabaseRepository.findAll();
    }

    public void delete(Long id) {
        gardenDatabaseRepository.delete(id);
    }
}
```


## Testing Hibernate Database Repository

I didn't share the GardenDatabaseRepository above because it simply calls through to the CrudRespository, but having the Flower associated on the Garden, we want to make sure that deletion of a Garden cascades to deletion of its flowers. This means we need to add tests for the GardenDatabaseRepository.

```
@RunWith(SpringRunner.class)
@SpringBootTest
@ActiveProfiles({"test"})
@Transactional
public class GardenDatabaseRepositoryTest {

  @Autowired
  FlowerDatabaseRepository flowerDatabaseRepository;

  GardenDatabaseRepository subject;

  @Autowired
  EntityManager entityManager;

  @Test
  public void delete_shouldDeleteAssociatedFlowers() throws Exception {
    Flower flower1 = new Flower();
    Flower flower2 = new Flower();

    Garden garden = Garden.builder().name("Community Flower Garden").build;
    garden.addFlowers(flower1, flower2);

    entityManager.persist(garden);

    Iterable<Flower> flowers = flowerDatabaseRepository.findAll();
    assertThat(flowers).hasSize(2);

    subject.delete(garden.getId());

    Iterable<Flower> flowersAfterDelete = flowerDatabaseRepository.findAll();
    assertThat(flowersAfterDelete).hasSize(0);
  }
}
```
Notice that we call `.persist` on the EntityManager in order to save the created flowers in the test database. This is because we want to make sure the database repository returns the objects we expect. Without the `.persist` call, the created flowers would not really be created in the database, but in memory. Hibernate does this to be efficient. That works well in your code, but not in test.

You'll also notice that we had to explicitly instantiate flowers to be added to the garden. With Hibernate, you have to modify the exact instance of the model you want. If we wanted to set attributes on those flowers, we would have needed `flower1.setColor("red")`. Moving away from Hibernate allows us create objects with a builder instead of making the object and then setting attributes on them. This makes our tests easier to read.

Because most of the GardenDatabaseRepository functions are from the extended CRUD Repository, we did not write a lot of tests for custom functions, but when we moved away from Hibernate, we needed to write a lot more tests. Here is an example of a `GardenRepositoryTest`:

```
@RunWith(SpringRunner.class)
class GardenRepositoryTest {
    /* Autowiring of all the components here, left out for brevity */

    private GardenRepository subject;

    @Test
    public void findAll_shouldReturnAllTheGardens() throws Exception {....}

    public void findOne_shouldReturnTheSpecifiedGarden() throws Exception {....}
    public void create_shouldSaveGardenAndReturnResult() throws Exception {....}
    public void create_whenDuplicateName_shouldReturnFailedResult() throws Exception {....}
    public void delete_whenSuccessful_shouldReturnSuccess throws Exception {....}
    public void delete_shouldDeleteAssociatedFlowers() throws Exception {....}
    public void delete_whenFailed_shouldReturnFailedResult() throws Exception {....}

    /* Other tests as needed */
}
```

## The Replacement JDBC Repository

To replace Hibernate, we create a GardenJdbcRepository that calls to a GardenJdbcPersister. The persister is where we make our sql calls.

### GardenJdbcRepository

```
@Service
public class GardenJdbcRepository implements GardenRepository {
  private final GardenJdbcPersister gardenJdbcPersister;

  GardenJdbcRepository(GardenJdbcPersister gardenJdbcPersister) {
    this.gardenJdbcPersister = gardenJdbcPersister;
  }

  public Optional<Garden> findOne(long gardenId) {
    return gardenJdbcPersister.findOne(gardenId);
  }

  public Result<Long, RepositoryFailure> create(Garden garden) {
    try {
      final long createGardenId = gardenJdbcPersister.create(garden);
      return Result.success(createGardenId);
    } catch (RepositoryException e) {
      return RepositoryFailureFactory.failure(e, garden);
    }
  }

  // Other functions like findAll, delete, etc....
}
```

### GardenJdbcPersister

```
@Component
public class GardenJdbcPersister {
  private final JdbcTemplate = jdbcTemplate;
  private final DbLastInsertedIdProvider dbLastInsertedIdProvider;

  public GardenJdbcPersister(JdbcTemplate jdbcTemplate, DbLastInsertedIdProvider dbLastInsertedIdProvider) {
    this.jdbcTemplate = jdbcTemplate;
    this.dbLastInsertedIdProvider = dbLastInsertedIdProvider;
  }

  public Optional<Garden> findOne(long id) {
    final Garden garden = jdbcTemplate.queryForObject("select * from gardens where id = ?", new GardenRowMapper(), id);

    return Optional.of(garden;)
  }

  public long create(Garden garden) throws RepositoryException {
    updateSingleRecord(
      "insert into gardens (name) " + "values (?)",
      garden.getName());

    return dbLastInsertedIdProvider.lastInsertedId();
  }

  // Other functions eg. findAll, delete, etc....

  private class GardenRowMapper implements RowMapper<Garden> {
    @Override
    public Garden mapRow(ResultSet rs, int rowNum) throws SQLException {
        return Garden.builder()
                .id(rs.getLong("id"))
                .name(rs.getString("name"))
                .build();
    }
  }
}
```

One interesting tidbit above is the DbLastInsertedIdProvider. When you successfully create or update an entry in sql, sql does not return the id of the object. We wrote a function that uses the sql `"select LAST_INSERT_ID()"` so that we can return the id of the object created/updated.

We also made a FlowerJdbcRepository and FlowerJdbcPersister in the same vein.

## Cleaner Garden and Flower Models

Now that Hibernate is out of our code, our model code is simplified.

### Garden Model, post-Hibernate
```
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity(name = "gardens")
public class Garden implements NameableEntity {
  @Id
  private Long id;
  private String name;
}
```

### Flower Model, post-Hibernate
```
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@EqualsAndHashCode
public class Flower {
  private Long id;
  private ZonedDateTime createdAt;
  private Long gardenId;
}
```

## Pros/Cons of the transition

Honestly, this was a lot of work. We were lucky that the app we were converting away from Hibernate was small. One great part of this re-organization is that it gave us a chance to re-think the structure of our app and spot some code cycles that we needed to untangle. We also discovered places where we depended on Hibernate's behavior instead of thinking through our app's architecture.


## Would we do it again?

If we were to do it again, we'd want to try the [Spring Data JDBC generic DAO implementation](https://github.com/jirutka/spring-data-jdbc-repository) mentioned above. For future apps, we will continue to use Hibernate and work around the testing issues, as unpleasant as they are. We will keep our newly JDBC-ed app and watch for ease of refactoring/adding new features and see how maintenance of the app goes.

If you are interested in removing Hibernate and replacing it with something lower-level like JDBC, start with a couple of fairly simple models and work from there.
