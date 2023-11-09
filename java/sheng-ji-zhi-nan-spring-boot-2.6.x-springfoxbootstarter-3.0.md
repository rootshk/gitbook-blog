# \[升级指南] Spring Boot 2.6.x springfox-boot-starter 3.0

* 升级Spring Boot 2.6.x 运行后报错

```
Cannot invoke "org.springframework.web.servlet.mvc.condition.PatternsRequestCondition.getPatterns()" because "this.condition" is null
```

* 解决方案

```
1. 创建文件 WebMvcRequestHandlerProvider.java

package springfox.documentation.spring.web.plugins;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
import org.springframework.context.annotation.Conditional;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.servlet.mvc.method.RequestMappingInfo;
import org.springframework.web.servlet.mvc.method.RequestMappingInfoHandlerMapping;
import springfox.documentation.RequestHandler;
import springfox.documentation.spi.service.RequestHandlerProvider;
import springfox.documentation.spring.web.OnServletBasedWebApplication;
import springfox.documentation.spring.web.WebMvcRequestHandler;
import springfox.documentation.spring.web.readers.operation.HandlerMethodResolver;

import javax.servlet.ServletContext;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;
import java.util.function.Function;
import java.util.stream.Collectors;
import java.util.stream.StreamSupport;

import static java.util.stream.Collectors.*;
import static springfox.documentation.builders.BuilderDefaults.*;
import static springfox.documentation.spi.service.contexts.Orderings.*;
import static springfox.documentation.spring.web.paths.Paths.*;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
@Conditional(OnServletBasedWebApplication.class)
public class WebMvcRequestHandlerProvider implements RequestHandlerProvider {
    private final List<RequestMappingInfoHandlerMapping> handlerMappings;
    private final HandlerMethodResolver methodResolver;
    private final String contextPath;

    @Autowired
    public WebMvcRequestHandlerProvider(Optional<ServletContext> servletContext, HandlerMethodResolver methodResolver,
                                        List<RequestMappingInfoHandlerMapping> handlerMappings) {
        this.handlerMappings = handlerMappings.stream().filter(mapping -> Objects.isNull(mapping.getPatternParser())).collect(Collectors.toList());
        this.methodResolver = methodResolver;
        this.contextPath = servletContext
                .map(ServletContext::getContextPath)
                .orElse(ROOT);
    }

    @Override
    public List<RequestHandler> requestHandlers() {
        return nullToEmptyList(handlerMappings).stream()
                .filter(requestMappingInfoHandlerMapping ->
                        !("org.springframework.integration.http.inbound.IntegrationRequestMappingHandlerMapping"
                                .equals(requestMappingInfoHandlerMapping.getClass()
                                        .getName())))
                .map(toMappingEntries())
                .flatMap((entries -> StreamSupport.stream(entries.spliterator(), false)))
                .map(toRequestHandler())
                .sorted(byPatternsCondition())
                .collect(toList());
    }

    private Function<RequestMappingInfoHandlerMapping,
            Iterable<Map.Entry<RequestMappingInfo, HandlerMethod>>> toMappingEntries() {
        return input -> input.getHandlerMethods()
                .entrySet();
    }

    private Function<Map.Entry<RequestMappingInfo, HandlerMethod>, RequestHandler> toRequestHandler() {
        return input -> new WebMvcRequestHandler(
                contextPath,
                methodResolver,
                input.getKey(),
                input.getValue());
    }
}
```
