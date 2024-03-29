# day01
## *Swagger*
**生成接口文档，在线接口调试**
### 1. *导入依赖*
```xml
        <dependency>
            <groupId>com.github.xiaoymin</groupId>
            <artifactId>knife4j-spring-boot-starter</artifactId>
            <version>3.0.2</version>
        </dependency>
```
### 2. *WebMvcConfiguration配置类中加入knife4j相关配置*
```java
    @Bean
    public Docket docket() {
        ApiInfo apiInfo = new ApiInfoBuilder()
                .title("苍穹外卖项目接口文档")
                .version("2.0")
                .description("苍穹外卖项目接口文档")
                .build();
        Docket docket = new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo)
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.sky.controller"))
                .paths(PathSelectors.any())
                .build();
        return docket;
    }
```
### 3. *设置静态资源映射*
```java
    protected void addResourceHandlers(ResourceHandlerRegistry registry) { 
        registry.addResourceHandler("/doc.html").addResourceLocations("classpath:/META-INF/resources/");
        registry.addResourceHandler("/webjars/**").addResourceLocations("classpath:/META-INF/resources/webjars/");
    }
```
### 4. *常用注解*
注解|说明
-|-
@Api(tags="")|用在类上,例如Controller,表明对类的说明
@APiModel|用在类上,例如Entity,DTO,VO
@ApiModelProperty|用在属性上,描述属性信息
@ApiOperation(value="")|用在方法上,例如Controller的方法,说明方法的用途,作用
# day02
## *新增员工*
### 代码开发
```word
当前端提交的数据与实体类中对应的属性差别比较大时，建议使用DTO来封装数据
DTO与实体类对象属性拷贝
```
1. Controller
```java
    @PostMapping
    @ApiOperation("新增员工")
    public Result save(@RequestBody EmployeeDTO employeeDTO){
        log.info("新增员工:{}",employeeDTO);
        employeeService.save(employeeDTO);
        return Result.success();
    }
```
2. Service
```java
    public void save(EmployeeDTO employeeDTO) {
        Employee employee=new Employee();
        BeanUtils.copyProperties(employeeDTO,employee);
        employee.setStatus(StatusConstant.ENABLE);
        employee.setPassword(DigestUtils.md5DigestAsHex(PasswordConstant.DEFAULT_PASSWORD.getBytes()));
        employee.setCreateTime(LocalDateTime.now());
        employee.setUpdateTime(LocalDateTime.now());
        //TODO 后期需要修改为创建人id和登录用户id
        employee.setCreateUser(10L);
        employee.setUpdateUser(10L);
        employeeMapper.insert(employee);
    }
```
3. Mapper
```java
    @Insert("insert into employee(name, username, password, phone, sex, id_number, status, create_time, update_time, create_user, update_user)" +
            "values " +
            "(#{name},#{username},#{password},#{phone},#{sex},#{idNumber},#{createTime},#{updateTime},#{createUser},#{updateUser},#{status})")
    void insert(Employee employee);
```
### 功能测试
- 功能测试方式:
+ 通过接口文档测试
+ 通过前后端联调测试
### 代码完善
+ 数据库唯一约束报错没有处理
-GlobalExceptionHandler
```java
@ExceptionHandler
    public Result exceptionHandler(SQLIntegrityConstraintViolationException ex){
        String message=ex.getMessage();
        if(message.contains("Duplicate entry")) {
            String[] split = message.split(" ");
            String username = split[2];
            String msg=username+MessageConstant.ALREADY_EXISTS;
            return Result.error(msg);
        }else {
            return Result.error(MessageConstant.UNKNOWN_ERROR);
        }
    }
```
+ setCreateUser和setUpdateUser设置为固定值，没有处理
- ThreadLocal jwt将用户id存入，Controller和Service再取出(**共享**)
```java
//jwt拦截器
Long empId = Long.valueOf(claims.get(JwtClaimsConstant.EMP_ID).toString());
BaseContext.setCurrentId(empId);
```
-----------
```java
//ServiceImpl
        employee.setCreateUser(BaseContext.getCurrentId());
        employee.setUpdateUser(BaseContext.getCurrentId());
```
## *员工分页查询* 
### 代码实现
1. Controller
```java
@GetMapping("/page")
    @ApiOperation("员工分页查询")
    public Result<PageResult> page(EmployeePageQueryDTO employeePageQueryDTO){
        log.info("员工分页查询,参数为:{}",employeePageQueryDTO);
        PageResult pageResult=employeeService.pageQuery(employeePageQueryDTO);
        return Result.success(pageResult);
    }
```
2. ServiceImpl
```java
public PageResult pageQuery(EmployeePageQueryDTO employeePageQueryDTO) {

        PageHelper.startPage(employeePageQueryDTO.getPage(),employeePageQueryDTO.getPageSize());
        Page<Employee>page=employeeMapper.pageQuery(employeePageQueryDTO);
        long total = page.getTotal();
        List<Employee> records = page.getResult();
        return new PageResult(total,records);
    }
```
3. Mapper
```xml
        <dependency>
            <groupId>com.github.pagehelper</groupId>
            <artifactId>pagehelper-spring-boot-starter</artifactId>
            <version>1.3.0</version>
        </dependency>
```
---------
```java
Page<Employee> pageQuery(EmployeePageQueryDTO employeePageQueryDTO);
```
--------
```xml
<mapper namespace="com.sky.mapper.EmployeeMapper">
    <select id="pageQuery" resultType="com.sky.entity.Employee">
        select * from employee
        <where>
            <if test="name!=null and name!= ''">
                and name like concat('%',#{name},'%')
            </if>
        </where>
        order by create_time desc
    </select>
</mapper>
```
### 代码完善
1. 数据库返回的日期格式显示有问题
+ 方法一:在属性上加入注解@JsonFormat(pattern="yyyy-MM-dd HH:mm:ss")
+ 方法二:在WebMvcConfiguration中扩展SpringMvc的消息转换器，统一对日期类型进行格式化处理
```java
    protected void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        log.info("扩展消息转换器");
        MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter();
        converter.setObjectMapper(new JacksonObjectMapper());
        converters.add(0,converter);
    }
```
## *启用禁用员工账号*
+ Shift+Alt+上下 
### 代码实现
1. Controller
```java
    @PostMapping("/status/{status}")
    @ApiOperation("启用禁用员工账号")
    public Result startOrStop(@PathVariable Integer status,Long id){
        log.info("启用禁用员工账号:{},{}",status,id);
        employeeService.startOrStop(status,id);
        return Result.success();
    }
```
2. ServiceImpl
```java
    public void startOrStop(Integer status, Long id) {

        Employee employee = Employee.builder()
                .status(status)
                .id(id).build();
        employeeMapper.update(employee);
    }
```
3. Mapper
```xml
        <dependency>
            <groupId>com.github.pagehelper</groupId>
            <artifactId>pagehelper-spring-boot-starter</artifactId>
            <version>1.3.0</version>
        </dependency>
```
---------
```java
void update(Employee employee);
```
--------
```xml
    <update id="update" parameterType="Employee">
        update employee
        <set>
            <if test="name !=null">name=#{name},</if>
            <if test="username !=null">username=#{username},</if>
            <if test="password!=null">password=#{password},</if>
            <if test="phone!=null">phone=#{phone},</if>
            <if test="sex!=null">sex=#{sex},</if>
            <if test="idNumber!=null">id_Number=#{name},</if>
            <if test="updateTime!=null">update_Time=#{updateTime},</if>
            <if test="updateUser!=null">update_User=#{updateUser},</if>
            <if test="status!=null">status=#{status},</if>
        </set>
        where id =#{id}
    </update>
```
## *编辑员工*
+ 查询和修改
### 开发代码
+ 查询
1. Controller
```java
    @GetMapping("/{id}")
    @ApiOperation("根据id查询员工信息")
    public Result<Employee> getById(@PathVariable long id){
        Employee employee=employeeService.getById(id);
        return Result.success(employee);
    }
```
2. Service
```java
    public Employee getById(long id) {
        Employee employee=employeeMapper.getById(id);
        employee.setPassword("****");
        return employee;
    }
```
3. Mapper
```java
    @Select("select * from employee where id=#{id};")
    Employee getById(long id);
```
+ 修改
1. Controller
```java
    @PutMapping
    @ApiOperation("编辑员工信息")
    public Result update(@RequestBody EmployeeDTO employeeDTO) {
        log.info("编辑员工信息:{}",employeeDTO);
        employeeService.update(employeeDTO);
        return Result.success();
    }
```
2. Service
```java
    public void update(EmployeeDTO employeeDTO) {
        Employee employee=new Employee();
        BeanUtils.copyProperties(employeeDTO,employee);
        employee.setUpdateTime(LocalDateTime.now());
        employee.setUpdateUser(BaseContext.getCurrentId());
        employeeMapper.update(employee);
    }
```
3. Mapper
```java
    void update(Employee employee);
```