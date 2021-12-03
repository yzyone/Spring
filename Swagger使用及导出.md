
# Swagger使用及导出 #

## 1、依赖 ##

```
<swagger.version>2.9.2</swagger.version>

<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>${swagger.version}</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>${swagger.version}</version>
</dependency>
```

## 2、SwaggerConfig ##

```
@Configuration
@EnableSwagger2
@Profile({"dev", "test"})
public class Swagger2Config {
    @Autowired
    private SignConfig appKeyConfig;

    @Bean
    public Docket createRestApi() {
        List<Parameter> pars = new ArrayList<>();
        if (appKeyConfig.isSignOn()) {
            //添加header
            ParameterBuilder timestamp = new ParameterBuilder();
            timestamp.name("timestamp").description("时间戳").modelRef(new ModelRef("string")).parameterType("header").required(true).build();

            ParameterBuilder sign = new ParameterBuilder();
            sign.name("sign").description("签名").modelRef(new ModelRef("string")).parameterType("header").required(true).build();

            ParameterBuilder appId = new ParameterBuilder();
            appId.name("appId").description("appId").modelRef(new ModelRef("string")).parameterType("header").required(true).build();

            pars.add(timestamp.build());
            pars.add(sign.build());
            pars.add(appId.build());
        }

        Docket docket = new Docket(DocumentationType.SWAGGER_2);

        docket.apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.withClassAnnotation(Api.class))//这是注意的代码
                .paths(PathSelectors.any())
                .build().globalOperationParameters(pars);
        return docket;
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("xxxxx接口文档")
                .termsOfServiceUrl("https://www.xxxxx.com")
                .version("1.0")
                .build();
    }
}
```

## 3、注解 ##

controller

	@Api(tags = "xxxxxxxAPI")

controller 方法

	@ApiOperation("xxxxxx接口")

VO、DTO内属性

	@ApiModelProperty(value = "id", example = "56449e5fb3a39283af2e1cbe")
	private String id;

## 4、导出html、pdf、markDown ##

配置pom后：

1. 生成markdown、ASCIIDOC，执行mvn swagger2markup:convertSwagger2markup
1. 生成 html 、pdf，执行1后，执行mvn generate-resources
1. pdf中文显示存在问题

```
<!-- 此插件生成markdown、ASCIIDOC、wiki格式-->
            <!--  执行命令 mvn swagger2markup:convertSwagger2markup-->
            <plugin>
                <groupId>io.github.swagger2markup</groupId>
                <artifactId>swagger2markup-maven-plugin</artifactId>
                <version>1.3.7</version>
                <configuration>
                    <!-- api-docs访问url -->
                    <swaggerInput>http://localhost:8099/food/v2/api-docs</swaggerInput>
                    <!-- 生成为单个文档，输出路径-->
                    <outputFile>src/docs/api</outputFile>
                    <!-- 生成为多个文档，输出路径 -->
                    <!--<outputDir>src/docs/</outputDir>-->
                    <config>
                        <!-- wiki格式文档 -->
                        <!--<swagger2markup.markupLanguage>CONFLUENCE_MARKUP</swagger2markup.markupLanguage>-->
                        <!-- ascii格式文档 -->
                        <swagger2markup.markupLanguage>ASCIIDOC</swagger2markup.markupLanguage>
                        <!-- markdown格式文档 -->
                        <!--<swagger2markup.markupLanguage>MARKDOWN</swagger2markup.markupLanguage>-->
                        <swagger2markup.pathsGroupedBy>TAGS</swagger2markup.pathsGroupedBy>
                    </config>
                </configuration>
            </plugin>
            <!--此插件生成HTML和PDF-->
            <!-- 执行命令 mvn generate-resources -->
            <plugin>
                <groupId>org.asciidoctor</groupId>
                <artifactId>asciidoctor-maven-plugin</artifactId>
                <version>2.1.0</version>
                <!-- Include Asciidoctor PDF for pdf generation -->
                <dependencies>
                    <dependency>
                        <groupId>org.asciidoctor</groupId>
                        <artifactId>asciidoctorj-pdf</artifactId>
                        <version>1.5.4</version>
                    </dependency>
                    <dependency>
                        <groupId>org.jruby</groupId>
                        <artifactId>jruby-complete</artifactId>
                        <version>9.2.17.0</version>
                    </dependency>
                </dependencies>
                <!-- Configure generic document generation settings -->
                <configuration>
                    <sourceDirectory>src/docs</sourceDirectory>
                    <!-- <sourceHighlighter>coderay</sourceHighlighter>-->
                    <attributes>
                        <toc>left</toc>
                    </attributes>
                </configuration>
                <!-- Since each execution can only handle one backend, run
                     separate executions for each desired output type -->
                <executions>
                    <execution>
                        <id>output-pdf</id>
                        <phase>generate-resources</phase>
                        <goals>
                            <goal>process-asciidoc</goal>
                        </goals>
                        <configuration>
                            <backend>pdf</backend>
                            <outputDirectory>src/docs/pdf/</outputDirectory>
                        </configuration>
                    </execution>
                    <execution>
                        <id>output-html</id>
                        <phase>generate-resources</phase>
                        <goals>
                            <goal>process-asciidoc</goal>
                        </goals>
                        <configuration>
                            <backend>html5</backend>
                            <outputDirectory>src/docs/html/</outputDirectory>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
```

作者：TheUnforgiven

链接：https://www.jianshu.com/p/f8ca5fa9a5a8

来源：简书

著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。