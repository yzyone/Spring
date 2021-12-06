# Swagger学习（二、配置swagger） #

基于上一篇

其实只是在SwaggerConfig.class中配置docket，apiInfo

```
@Configuration   //变成配置文件
@EnableSwagger2  //开启swagger2
public class SwaggerConfig {
    @Bean  //配置swagger的docket的bean实例
    public Docket docket(){
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo());
    }
    //配置swagger信息的ApiInfo
    private ApiInfo apiInfo(){
        //作者的联系方式
        Contact contact = new Contact("xiaowei","https://www.baidu.com","1102356056@qq.com");
        return new ApiInfo(
                "xiaowei hello",
                "有一种想见不敢见的伤痛 有一种爱还埋藏在我心中",
                "v0.1",
                "ttps://www.baidu.com",
                contact,
                "Apache 2.0",
                "http://www.apache.org/licenses/LICENSE-2.0",
                new ArrayList()
        );
    }
}
```

效果