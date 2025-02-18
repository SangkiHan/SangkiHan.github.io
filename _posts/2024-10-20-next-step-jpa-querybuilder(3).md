---
title: EntityManager 구현
author: SangkiHan
date: 2024-10-20 11:20:00 +0900
categories: [NEXT-STEP, JPA]
tags: [NEXT-STEP]
---
------------

[Github Repo](https://github.com/SangkiHan/jpa-query-builder)

기존에는 H2QueryBuilderDML, H2QueryBuilderDDL 내부에서 DMLColumnData와 DDLColumnData까지 각각 내부 데이터를 검증하는 로직이 포함되어 있었다.  
하지만 각 클래스안에 너무 많은 책임이 있다고 생각하여 클래스들을 감싸고 각 쿼리 메소드별로 클래스를 나눠 책임을 분산하였다.

### DDLBuilderData
DDLBuilderData 객체를 생성하여 DDLColumnData의 검증과 데이터 관리의 책임을 할당했다.
``` java
public class DDLBuilderData {

    private final static String ID_ANNOTATION_OVER_ONE = "@Id 어노테이션은 한개를 초과할수 없습니다.";
    private final static String NOT_EXIST_ENTITY_ANNOTATION = "@Entity 어노테이션이 존재하지 않습니다.";

    private final static String PRIMARY_KEY = " PRIMARY KEY";
    private final static String NOT_NULL = " NOT NULL";
    private final static String AUTO_INCREMENT = " AUTO_INCREMENT";
    private final static String COMMA = ", ";
    private final static String BLANK = " ";

    private final String tableName;
    private final List<DDLColumnData> columns;
    private final DB db;

    private <T> DDLBuilderData(Class<T> clazz, DB db) {
        confirmEntityAnnotation(clazz);
        this.db = db;
        this.tableName = getTableName(clazz);
        this.columns = getDDLColumnData(clazz);
    }

    public static <T> DDLBuilderData createDDLBuilderData(Class<T> clazz, DB db) {
        return new DDLBuilderData(clazz, db);
    }

    // 테이블 열 정의 생성
    public String getColumnDefinitions() {
        return this.columns.stream()
                .map(column -> {
                    String definition = column.columnName() + BLANK + column.columnDataType();
                    // primary key인 경우 "PRIMARY KEY" 추가
                    if (column.isNotNull()) definition += NOT_NULL; //false면 NOT_NULL 조건 추가
                    if (column.isAutoIncrement()) definition += AUTO_INCREMENT; //true면 AutoIncrement 추가
                    if (column.isPrimaryKey()) definition += PRIMARY_KEY; //PK면 PK조건 추가
                    return definition;
                })
                .collect(Collectors.joining(COMMA));
    }

    public String getTableName() {
        return tableName;
    }

    //변수들의 정보를 가져온다.
    private List<DDLColumnData> getDDLColumnData(Class<?> entityClass) {
        List<DDLColumnData> columnData = Arrays.stream(entityClass.getDeclaredFields())
                .map(this::createTableDDLColumnData)
                .filter(Objects::nonNull)
                .collect(Collectors.toList());

        confirmIdAnnotationOverTwo(columnData);
        return columnData;
    }

    //테이블에 생성될 필드(컬럼)들을 생성한다.
    private DDLColumnData createTableDDLColumnData(Field field) {
        if (field.isAnnotationPresent(Id.class)) {
            return getPrimaryKey(field);
        }
        return getColumnAnnotationData(field);
    }

    //Id 어노테이션을 primarykey로 가져온다.
    private DDLColumnData getPrimaryKey(Field field) {
        return DDLColumnData.createPk(
                field.getName(),
                field.getType(),
                confirmGeneratedValueAnnotation(field),
                this.db
        );
    }

    //Column 어노테이션 여부를 확인하여 변수의 컬럼타입을 가져온다.
    private DDLColumnData getColumnAnnotationData(Field field) {
        if (field.isAnnotationPresent(Transient.class)) {
            return null;
        }
        String columnName = field.getName();
        boolean isNullable = true;

        if (field.isAnnotationPresent(Column.class)) {
            Column column = field.getAnnotation(Column.class);
            columnName = column.name().isEmpty() ? columnName : column.name();
            return DDLColumnData.createColumn(
                    columnName,
                    field.getType(),
                    !column.nullable(),
                    this.db
            );
        }

        return DDLColumnData.createColumn(
                columnName,
                field.getType(),
                !isNullable,
                this.db
        );
    }

    // Entity에 @Id가 2개 이상은 아닐지 확인한다.
    private void confirmIdAnnotationOverTwo(List<DDLColumnData> DDLColumnDataList) {
        long primaryKeyCount = DDLColumnDataList.stream()
                .filter(DDLColumnData::isPrimaryKey)
                .count();

        if (primaryKeyCount >= 2) {
            throw new IllegalArgumentException(ID_ANNOTATION_OVER_ONE); // 2개 이상이면 예외 발생
        }
    }

    //GeneratedValue 어노테이션 전략을 확인한다.
    private boolean confirmGeneratedValueAnnotation(Field field) {
        if (!field.isAnnotationPresent(GeneratedValue.class)) {
            return false;
        }
        GeneratedValue generatedValue = field.getAnnotation(GeneratedValue.class);
        return generatedValue.strategy() == GenerationType.IDENTITY;
    }


    //Entity 어노테이션 여부를 확인한다.
    private void confirmEntityAnnotation(Class<?> entityClass) {
        if (!entityClass.isAnnotationPresent(Entity.class)) {
            throw new IllegalArgumentException(NOT_EXIST_ENTITY_ANNOTATION);
        }
    }

    //Table 어노테이션 여부를 확인한다.
    private String getTableName(Class<?> entityClass) {
        if (entityClass.isAnnotationPresent(Table.class)) {
            Table table = entityClass.getAnnotation(Table.class);
            return table.name();
        }
        return entityClass.getSimpleName();
    }
}
```

### DMLBuilderData
DMLBuilderData 객체를 생성하여 DMLColumnData의 검증과 데이터 관리의 책임을 할당했다.

``` java
public class DMLBuilderData {

    private final static String PK_NOT_EXIST_MESSAGE = "PK 컬럼을 찾을 수 없습니다.";
    private final static String NOT_EXIST_ENTITY_ANNOTATION = "@Entity 어노테이션이 존재하지 않습니다.";
    private final static String GET_FIELD_VALUE_ERROR_MESSAGE = "필드 값을 가져오는 중 에러가 발생했습니다.";
    private final static String COMMA = ", ";
    private final static String EQUALS = "=";

    private final String tableName;
    private final List<DMLColumnData> columns;
    private Object id;

    private <T> DMLBuilderData(Class<T> clazz) {
        confirmEntityAnnotation(clazz);
        this.tableName = getTableName(clazz);
        this.columns = getEntityColumnData(clazz);
    }

    private DMLBuilderData(Object entityInstance) {
        confirmEntityAnnotation(entityInstance.getClass());
        this.tableName = getTableName(entityInstance.getClass());
        this.columns = getInstanceColumnData(entityInstance);
        this.id = getPkValue(this.columns);
    }

    private <T> DMLBuilderData(Class<T> clazz, Object id) {
        confirmEntityAnnotation(clazz);
        this.tableName = getTableName(clazz);
        this.columns = getEntityColumnData(clazz);
        this.id = id;
    }

    public static <T> DMLBuilderData createDMLBuilderData(Class<T> clazz) {
        return new DMLBuilderData(clazz);
    }

    public static DMLBuilderData createDMLBuilderData(Object entityInstance) {
        return new DMLBuilderData(entityInstance);
    }

    public static <T> DMLBuilderData createDMLBuilderData(Class<T> clazz, Object id) {
        return new DMLBuilderData(clazz, id);
    }

    public String getTableName() {
        return tableName;
    }

    public List<DMLColumnData> getColumns() {
        return columns;
    }

    public Object getId() {
        return id;
    }

    public String wrapString() {
        return (this.id instanceof String) ? StringUtil.wrapSingleQuote(this.id) : String.valueOf(this.id);
    }

    // 테이블 열 정의 생성
    public String getColumnDefinitions() {
        return this.columns.stream()
                .filter(column -> !column.isPrimaryKey())
                .map(column -> column.getColumnName() + EQUALS + column.getColumnValueByType())
                .collect(Collectors.joining(COMMA));
    }

    // 테이블 컬럼명 생성
    public String getColumnNames() {
        return this.columns.stream()
                .map(DMLColumnData::getColumnName)
                .collect(Collectors.joining(COMMA));
    }

    //테이블 컬럼 Value 값들 생성
    public String getColumnValues() {
        return this.columns.stream()
                .map(dmlColumnData -> {
                    Object value = dmlColumnData.getColumnValue();
                    if (dmlColumnData.getColumnType() == String.class) { //데이터 타입이 String 이면 작은 따옴표로 묶어준다.
                        return StringUtil.wrapSingleQuote(value);
                    }
                    return String.valueOf(value);
                })
                .collect(Collectors.joining(COMMA));
    }

    //PkName를 가져온다.
    public String getPkName() {
        return this.columns.stream()
                .filter(DMLColumnData::isPrimaryKey)
                .map(DMLColumnData::getColumnName)
                .findFirst()
                .orElseThrow(() -> new RuntimeException(PK_NOT_EXIST_MESSAGE));
    }

    //Entity Class의 컬럼명과 컬럼데이터타입을 가져온다.
    private List<DMLColumnData> getEntityColumnData(Class<?> entityClass) {
        Field[] fields = entityClass.getDeclaredFields();
        List<DMLColumnData> DMLColumnDataList = new ArrayList<>();
        for (Field field : fields) {
            getEntityPrimaryKey(DMLColumnDataList, field);
            createDMLEntityColumnData(DMLColumnDataList, field);
        }
        return DMLColumnDataList;
    }

    //Entity 인스턴스의 컬럼명과 컬럼데이터타입, 컬럼데이터를 가져온다.
    private <T> List<DMLColumnData> getInstanceColumnData(T entityInstance) {
        Field[] fields = entityInstance.getClass().getDeclaredFields();
        List<DMLColumnData> DMLColumnDataList = new ArrayList<>();
        for (Field field : fields) {
            getInstancePrimaryKey(DMLColumnDataList, field, entityInstance);
            createDMLInstanceColumnData(DMLColumnDataList, field, entityInstance);
        }
        return DMLColumnDataList;
    }

    //Id 어노테이션을 primarykey로 가져온다.
    private void getEntityPrimaryKey(List<DMLColumnData> DMLColumnDataList, Field field) {
        if (field.isAnnotationPresent(Id.class)) {
            DMLColumnDataList.add(DMLColumnData.creatInstancePkColumn(field.getName(), field.getType()));
        }
    }

    //Id 어노테이션을 primarykey로 가져온다.
    private <T> void getInstancePrimaryKey(List<DMLColumnData> DMLColumnDataList, Field field, T entityInstance) {
        try {
            if (field.isAnnotationPresent(Id.class)) {
                field.setAccessible(true);
                DMLColumnDataList.add(DMLColumnData.creatEntityPkColumn(field.getName(), field.getType(), field.get(entityInstance)));
            }
        } catch (IllegalAccessException e) {
            throw new RuntimeException(GET_FIELD_VALUE_ERROR_MESSAGE + field.getName(), e);
        }
    }

    //Entity 내부 필드를 확인하여 필드명을 가져온다.
    private void createDMLEntityColumnData(List<DMLColumnData> DMLColumnDataList, Field field) {
        if (field.isAnnotationPresent(Transient.class) || field.isAnnotationPresent(Id.class))
            return; // @Transient인 경우 검증하지 않음

        String columnName = field.getName();

        if (field.isAnnotationPresent(Column.class)) {
            Column column = field.getAnnotation(Column.class);
            columnName = column.name().isEmpty() ? columnName : column.name();
        }

        DMLColumnDataList.add(DMLColumnData.createEntityColumn(columnName));
    }

    //인스턴스 내부 데이터를 확인하여 컬럼 데이터를 가져온다.
    private <T> void createDMLInstanceColumnData(List<DMLColumnData> DMLColumnDataList, Field field, T entityInstance) {
        if (field.isAnnotationPresent(Transient.class) || field.isAnnotationPresent(Id.class))
            return; // @Transient인 경우 검증하지 않음

        String columnName = field.getName();

        if (field.isAnnotationPresent(Column.class)) {
            Column column = field.getAnnotation(Column.class);
            columnName = column.name().isEmpty() ? columnName : column.name();
        }

        field.setAccessible(true);

        try {
            DMLColumnDataList.add(DMLColumnData.creatInstanceColumn(columnName, field.getType(), field.get(entityInstance)));
        } catch (IllegalAccessException e) {
            throw new RuntimeException(GET_FIELD_VALUE_ERROR_MESSAGE + field.getName(), e);
        }
    }

    //Entity 어노테이션 여부를 확인한다.
    private void confirmEntityAnnotation(Class<?> entityClass) {
        if (!entityClass.isAnnotationPresent(Entity.class)) {
            throw new IllegalArgumentException(NOT_EXIST_ENTITY_ANNOTATION);
        }
    }

    //Table 어노테이션 여부를 확인한다.
    private String getTableName(Class<?> entityClass) {
        if (entityClass.isAnnotationPresent(Table.class)) {
            Table table = entityClass.getAnnotation(Table.class);
            return table.name();
        }
        return entityClass.getSimpleName();
    }

    //PkValue를 가져온다.
    private Object getPkValue(List<DMLColumnData> DMLColumnDataList) {
        return DMLColumnDataList.stream()
                .filter(DMLColumnData::isPrimaryKey)
                .findFirst()
                .map(DMLColumnData::getColumnValue)
                .orElseThrow(() -> new IllegalArgumentException(PK_NOT_EXIST_MESSAGE));
    }

}
```

### builder별 클래스 생성하여 책임 분산
이전에는 DML, DDL별로 모든 쿼리가 한 클래스에 있어 너무 코드가 산만했다.

``` java
public class DeleteQueryBuilder {

    private final static String DELETE_BY_ID_QUERY = "DELETE FROM {tableName} WHERE {entityPkName} = {values};";
    private final static String TABLE_NAME = "{tableName}";
    private final static String VALUES = "{values}";
    private final static String ENTITY_PK_NAME = "{entityPkName}";

    public String buildQuery(DMLBuilderData dmlBuilderData) {
        return deleteByIdQuery(dmlBuilderData);
    }

    //delete 쿼리문을 생성한다.
    private String deleteByIdQuery(DMLBuilderData dmlBuilderData) {
        return DELETE_BY_ID_QUERY.replace(TABLE_NAME, dmlBuilderData.getTableName())
                .replace(ENTITY_PK_NAME, dmlBuilderData.getPkName())
                .replace(VALUES, String.valueOf(dmlBuilderData.wrapString()));
    }

}
```

``` java
public class InsertQueryBuilder {

    private final static String INSERT_QUERY = "INSERT INTO {tableName} ({columnNames}) VALUES ({values});";
    private final static String TABLE_NAME = "{tableName}";
    private final static String COLUMN_NAMES = "{columnNames}";
    private final static String VALUES = "{values}";

    //insert 쿼리를 생성한다. Insert 쿼리는 인스턴스의 데이터를 받아야함
    public String buildQuery(DMLBuilderData dmlBuilderData) {
        return insertQuery(dmlBuilderData);
    }

    //insert쿼리문을 생성한다.
    private String insertQuery(DMLBuilderData dmlBuilderData) {
        return INSERT_QUERY.replace(TABLE_NAME, dmlBuilderData.getTableName())
                .replace(COLUMN_NAMES, dmlBuilderData.getColumnNames())
                .replace(VALUES, dmlBuilderData.getColumnValues());
    }

}
```

``` java
public class SelectAllQueryBuilder {

    private final static String FIND_ALL_QUERY = "SELECT {columnNames} FROM {tableName};";
    private final static String TABLE_NAME = "{tableName}";
    private final static String COLUMN_NAMES = "{columnNames}";

    public String buildQuery(DMLBuilderData dmlBuilderData) {
        return findAllQuery(dmlBuilderData);
    }

    //findAll 쿼리문을 생성한다.
    private String findAllQuery(DMLBuilderData dmlBuilderData) {
        return FIND_ALL_QUERY.replace(TABLE_NAME, dmlBuilderData.getTableName())
                .replace(COLUMN_NAMES, dmlBuilderData.getColumnNames());
    }

}
```

``` java
public class SelectByIdQueryBuilder {

    private final static String FIND_BY_ID_QUERY = "SELECT {columnNames} FROM {tableName} WHERE {entityPkName} = {values};";
    private final static String TABLE_NAME = "{tableName}";
    private final static String COLUMN_NAMES = "{columnNames}";
    private final static String VALUES = "{values}";
    private final static String ENTITY_PK_NAME = "{entityPkName}";

    public String buildQuery(DMLBuilderData dmlBuilderData) {
        return findByIdQuery(dmlBuilderData);
    }

    //findAll 쿼리문을 생성한다.
    private String findByIdQuery(DMLBuilderData dmlBuilderData) {
        return FIND_BY_ID_QUERY.replace(TABLE_NAME, dmlBuilderData.getTableName())
                .replace(COLUMN_NAMES, dmlBuilderData.getColumnNames())
                .replace(ENTITY_PK_NAME, dmlBuilderData.getPkName())
                .replace(VALUES, String.valueOf(dmlBuilderData.wrapString()));
    }

}
```

``` java
public class UpdateQueryBuilder {

    private final static String UPDATE_BY_ID_QUERY = "UPDATE {tableName} SET {columnDefinitions} WHERE {entityPkName} = {values};";
    private final static String TABLE_NAME = "{tableName}";
    private final static String VALUES = "{values}";
    private final static String ENTITY_PK_NAME = "{entityPkName}";
    private final static String COLUMN_DEFINITIONS = "{columnDefinitions}";

    public String buildQuery(DMLBuilderData dmlBuilderData) {
        return updateByIdQuery(dmlBuilderData);
    }

    //update 쿼리를 생성한다.
    private String updateByIdQuery(DMLBuilderData dmlBuilderData) {
        // 최종 SQL 쿼리 생성
        return UPDATE_BY_ID_QUERY.replace(TABLE_NAME, dmlBuilderData.getTableName())
                .replace(COLUMN_DEFINITIONS, dmlBuilderData.getColumnDefinitions())
                .replace(ENTITY_PK_NAME, dmlBuilderData.getPkName())
                .replace(VALUES, String.valueOf(dmlBuilderData.wrapString()));
    }
}
```

### EntityManager

EntityManager는 주로 Java Persistence API (JPA)에서 사용되는 개념으로, 객체와 데이터베이스 간의 상호작용을 관리하는 인터페이스입니다. 이 객체는 데이터베이스에서 데이터를 저장, 조회, 수정 및 삭제하는 역할을 수행하며, ORM(Object-Relational Mapping) 프레임워크에서 엔티티의 수명 주기를 관리하는 데 중요한 기능을 합니다.  

EntityManager의 persist, find, findAll, update, remove를 구현하여 사용자가 위에 Builder들을 직접 가져다 사용하지 않고 EntityManager를 통해 각 기능을 제공해준다.

``` java
public interface EntityManager {

    <T> T find(Class<T> clazz, Long id);

    <T> List<T> findAll(Class<T> clazz);

    void persist(Object entityInstance);

    void update(Object entityInstance);

    void remove(Object entityInstance);

}
```

``` java
public class EntityManagerImpl implements EntityManager {

    private final static String DATA_NOT_EXIST_MESSAGE = "데이터가 존재하지 않습니다. : ";

    private final JdbcTemplate jdbcTemplate;

    public EntityManagerImpl(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public <T> T find(Class<T> clazz, Long id) {
        SelectByIdQueryBuilder queryBuilder = new SelectByIdQueryBuilder();
        return jdbcTemplate.queryForObject(queryBuilder.buildQuery(DMLBuilderData.createDMLBuilderData(clazz, id)), resultSet -> EntityMapper.mapRow(resultSet, clazz));
    }

    @Override
    public <T> List<T> findAll(Class<T> clazz) {
        SelectAllQueryBuilder queryBuilder = new SelectAllQueryBuilder();
        return jdbcTemplate.query(queryBuilder.buildQuery(DMLBuilderData.createDMLBuilderData(clazz)), resultSet -> EntityMapper.mapRow(resultSet, clazz));
    }

    @Override
    public void persist(Object entityInstance) {
        InsertQueryBuilder queryBuilder = new InsertQueryBuilder();
        jdbcTemplate.execute(queryBuilder.buildQuery(DMLBuilderData.createDMLBuilderData(entityInstance)));
    }

    @Override
    public void update(Object entityInstance) {
        confirmEntityDataExist(entityInstance);
        UpdateQueryBuilder queryBuilder = new UpdateQueryBuilder();
        jdbcTemplate.execute(queryBuilder.buildQuery(DMLBuilderData.createDMLBuilderData(entityInstance)));
    }

    @Override
    public void remove(Object entityInstance) {
        DeleteQueryBuilder queryBuilder = new DeleteQueryBuilder();
        jdbcTemplate.execute(queryBuilder.buildQuery(DMLBuilderData.createDMLBuilderData(entityInstance)));
    }

    //조회되는 데이터가 존재하는지 확인한다.
    private void confirmEntityDataExist(Object entityInstance) {
        SelectByIdQueryBuilder queryBuilder = new SelectByIdQueryBuilder();
        try {
            jdbcTemplate.queryForObject(
                    queryBuilder.buildQuery(DMLBuilderData.createDMLBuilderData(entityInstance)),
                    resultSet -> EntityMapper.mapRow(resultSet, entityInstance.getClass())
            );
        } catch (RuntimeException e) {
            throw new RuntimeException(DATA_NOT_EXIST_MESSAGE + entityInstance.getClass().getSimpleName());
        }
    }
}
```

### EntityMapper
EntityMapper를 통해 Entity를 조회를 하면 Object 타입으로 내려주는게 아니라 Entity클래스에 맞게 데이터를 세팅하여 응답해줘야한다.

``` java
public class EntityMapper {

    private final static String FAILED_GET_COLUMN = "컬럼 데이터를 가져오는데 실패했습니다.";
    private final static String FAILED_ACCESS_FIELD = "필드에 접근을 실패했습니다.";
    private final static String FAILED_CREATE_INSTANCE = "인스턴스를 생성하는데 실패하였습니다.";

    //입력 받은 Entity 에 맞게 자동으로 매핑한다.
    public static <T> T mapRow(ResultSet rs, Class<T> entityClass) {
        try {
            // 해당 클래스의 인스턴스 생성
            T entityInstance = entityClass.getDeclaredConstructor().newInstance();
            Field[] fields = entityClass.getDeclaredFields();

            for (Field field : fields) {
                confirmAnnotationSetColumnFieldName(field, rs, entityInstance);
            }
            return entityInstance;
        } catch (InstantiationException | IllegalAccessException | NoSuchMethodException |
                 InvocationTargetException e) {
            throw new RuntimeException(FAILED_CREATE_INSTANCE);
        }
    }

    // 인스턴스의 어노테이션을 검증하여 컬럼 데이터를 세팅해준다.
    private static <T> void confirmAnnotationSetColumnFieldName(Field field, ResultSet rs, T entityInstance) {
        if (field.isAnnotationPresent(Transient.class)) return;
        field.setAccessible(true);
        try {
            String columnName = field.getName();
            if (field.isAnnotationPresent(Column.class)) {
                Column column = field.getAnnotation(Column.class);
                columnName = column.name().isEmpty() ? columnName : column.name();
            }
            Object value = rs.getObject(columnName);
            field.set(entityInstance, value);
        } catch (SQLException e) {
            throw new RuntimeException(FAILED_GET_COLUMN);
        } catch (IllegalAccessException e) {
            throw new RuntimeException(FAILED_ACCESS_FIELD);
        }
    }

}
```

### Junit Test

#### EntityManagerTest 테스트
``` java
public class EntityManagerTest {

    private EntityManager em;
    private H2DBConnection h2DBConnection;
    private JdbcTemplate jdbcTemplate;

    @BeforeEach
    void setUp() throws SQLException {
        this.h2DBConnection = new H2DBConnection();
        this.jdbcTemplate = this.h2DBConnection.start();

        //테이블 생성
        CreateQueryBuilder queryBuilder = new CreateQueryBuilder();
        String createQuery = queryBuilder.buildQuery(DDLBuilderData.createDDLBuilderData(Person.class, DB.H2));

        jdbcTemplate.execute(createQuery);

        this.em = new EntityManagerImpl(jdbcTemplate);
    }

    //정확한 테스트를 위해 메소드마다 테이블 DROP 후 DB종료
    @AfterEach
    void tearDown() {
        DropQueryBuilder queryBuilder = new DropQueryBuilder();
        String dropQuery = queryBuilder.buildQuery(DDLBuilderData.createDDLBuilderData(Person.class, DB.H2));
        jdbcTemplate.execute(dropQuery);
        this.h2DBConnection.stop();
    }

    @DisplayName("Persist로 Person 저장 후 find로 조회한다.")
    @Test
    void findTest() {
        Person person = createPerson(1);
        this.em.persist(person);

        Person findPerson = this.em.find(Person.class, person.getId());

        assertThat(findPerson)
                .extracting("id", "name", "age", "email")
                .contains(1L, "test1", 29, "test@test.com");
    }

    @DisplayName("remove 실행한다.")
    @Test
    void removeTest() {
        Person person = createPerson(1);
        this.em.persist(person);
        this.em.remove(person);

        assertThatThrownBy(() -> this.em.find(Person.class, person.getId()))
                .isInstanceOf(RuntimeException.class)
                .hasMessage("Expected 1 result, got 0");
    }

    @DisplayName("update 실행한다.")
    @Test
    void updateTest() {
        Person person = createPerson(1);
        this.em.persist(person);

        person.changeEmail("changed@test.com");
        this.em.update(person);

        Person findPerson = this.em.find(Person.class, person.getId());

        assertThat(findPerson)
                .extracting("id", "name", "age", "email")
                .contains(1L, "test1", 29, "changed@test.com");
    }

    @DisplayName("update 실행할 시 존재하지 않은 데이터라면 예외를 발생시킨다.")
    @Test
    void updateThrowExceptionTest() {
        Person person = createPerson(1);

        assertThatThrownBy(() -> this.em.update(person))
                .isInstanceOf(RuntimeException.class)
                .hasMessage("데이터가 존재하지 않습니다. : Person");
    }

    private Person createPerson(int i) {
        return new Person((long) i, "test" + i, 29, "test@test.com");
    }
}
```