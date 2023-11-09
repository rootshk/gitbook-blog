# \[Server酱 Turbo版使用] Spring Boot 2.6异常处理增加Server酱消息通知

## 申请账号

[点击扫码申请Server Turbo](https://sct.ftqq.com/)

### 申请成功后在[https://sct.ftqq.com/sendkey](https://sct.ftqq.com/sendkey)可查看你自己的key，后面用于配置

## 代码示例

### 服务调用

```
package top.roothk.mall.service.impl;

import jodd.http.HttpRequest;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import top.roothk.mall.service.MessageService;

import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;

/**
 * @author hongkeng
 */
@Slf4j
@Service
public class MessageServiceImpl implements MessageService {

    @Value("${system.message.exception.server-turbo.key}")
    private String serverTurboKey;
    @Value("${spring.profiles.active:default}")
    private String activeProfile;

    private static final String SERVER_TURBO_URL = "https://sctapi.ftqq.com/%s.send";

    @Async
    @Override
    public void exception(String message) {
        if (!"prod".equals(activeProfile)) {
            return;
        }
        sendServerTurbo(serverTurboKey, "异常信息", message);
    }

    public static void sendServerTurbo(String serverTurboKey, String title, String message) {
        HttpRequest.get(String.format(SERVER_TURBO_URL, serverTurboKey))
                .connectionTimeout(60000)
                .timeout(60000)
                .charset("utf-8")
                .query("title", title)
                .query("desp", message)
                .send();
    }

}

```

### 全局异常处理

```
package top.roothk.mall.exception;

import com.alibaba.fastjson.JSON;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.TypeMismatchException;
import org.springframework.http.HttpStatus;
import org.springframework.validation.BindException;
import org.springframework.validation.ObjectError;
import org.springframework.web.HttpRequestMethodNotSupportedException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.MissingServletRequestParameterException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException;
import top.roothk.mall.ResultBean;
import top.roothk.mall.service.MessageService;

import javax.annotation.Resource;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.ByteArrayOutputStream;
import java.io.PrintStream;
import java.util.ArrayList;
import java.util.List;
import java.util.Objects;
import java.util.stream.Collectors;

/**
 * 服务器异常
 *
 * @author porridge
 */
@Slf4j
@RestControllerAdvice
public class RestExceptionController {

    @Resource
    private MessageService messageService;

    /**
     * 自定义异常类
     */
    @ExceptionHandler(value = ServiceException.class)
    public ResultBean<String> commonServiceException(HttpServletRequest req, HttpServletResponse resp, ServiceException e) {
        ResultBean<String> result = new ResultBean<>();
        result.setStatus(e.getRetCode());
        result.setMessage(e.getMessage());
        log.error("业务错误, {}", e.getMessage());
        messageService.exception(e.getMessage());
        return result;
    }

    /**
     * 未找到该方法或方法类型错误
     */
    @ExceptionHandler(value = {HttpRequestMethodNotSupportedException.class})
    public ResultBean<String> notFindException(HttpServletRequest req, HttpServletResponse resp, HttpRequestMethodNotSupportedException e) {
        resp.setStatus(404);
        log.error("未找到该地址: {}", req.getServletPath(), e);
        messageService.exception(e.getMessage());
        return new ResultBean<>(404, "未找到该地址");
    }

    @ExceptionHandler(value = {IllegalArgumentException.class})
    public ResultBean<String> illegalArgumentException(HttpServletRequest req, HttpServletResponse resp, IllegalArgumentException e) {
        log.error("IllegalArgumentException", e);
        messageService.exception(e.getMessage());
        return new ResultBean<>(400, e.getMessage());
    }

    /**
     * 参数验证失败 （注意：返回的 HTTP Status 是 400）
     */
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(value = {
            MethodArgumentNotValidException.class,
            MethodArgumentTypeMismatchException.class,
            MissingServletRequestParameterException.class,
            TypeMismatchException.class})
    public ResultBean<List<String>> validationException(HttpServletRequest req, HttpServletResponse resp, Exception e) {
        ResultBean<List<String>> result = new ResultBean<>();
        String msg = "参数验证失败";

        if (e instanceof MethodArgumentNotValidException exception) {
            List<String> errors = exception.getBindingResult()
                    .getFieldErrors()
                    .stream()
                    .map(x -> x.getField() + " 验证失败：" + x.getRejectedValue())
                    .collect(Collectors.toList());
            msg = JSON.toJSONString(errors);
        }

        if (e instanceof MethodArgumentTypeMismatchException exception) {
            msg = exception.getMessage();
        }
        log.info("参数验证失败 报警{} {} {}", "请求参数转换失败", JSON.toJSONString(result), e.getClass().toString());
        log.error("error", e);
        messageService.exception(e.getMessage());

        result.setStatus(400);
        result.setMessage(msg);
        result.setData(new ArrayList<>());
        resp.setStatus(400);
        return result;
    }

    @ExceptionHandler(value = {BindException.class})
    public ResultBean<List<String>> validationBindException(HttpServletRequest req, HttpServletResponse resp, BindException e) {
        resp.setStatus(400);
        ResultBean<List<String>> result = new ResultBean<>();
        result.setStatus(400);
        result.setMessage("类型验证错误");
        List<ObjectError> objectErrors = e.getAllErrors();
        List<String> errors = new ArrayList<>();
        for (ObjectError error : objectErrors) {
            errors.add(Objects.requireNonNull(error.getCodes())[1] + error.getDefaultMessage());
        }
        result.setData(errors);
        log.error("类型验证错误, {}", e.getMessage(), e);
        messageService.exception(e.getMessage());
        return result;
    }

    /**
     * 服务器通用异常
     */
    @ExceptionHandler(value = Exception.class)
    public ResultBean<String> handleException(HttpServletRequest req, HttpServletResponse resp, Exception e) {
        resp.setStatus(500);
        log.error("服务器异常: {}", e.getMessage(), e);
        messageService.exception(e.getMessage());
        return new ResultBean<>(500, "服务器异常: {}", e.getMessage());
    }

    /**
     * 获取错误信息完整信息
     *
     * @param e 错误
     * @return 错误信息
     */
    private String getErrorMessage(Exception e) {
        ByteArrayOutputStream b = new ByteArrayOutputStream();
        e.printStackTrace(new PrintStream(b));
        return b.toString();
    }

}

```
