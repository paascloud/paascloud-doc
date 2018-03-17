## 异常处理
参数抛指定的参数异常， 业务异常必须抛出指定编码。
正例：


```
throw new UacBizException(ErrorCodeEnum.UAC10011021);
```


如有业务编码 需要抛出指定业务编码


```
throw new UacBizException(ErrorCodeEnum.UAC10013002, menuId);
```

ErrorCodeEnum枚举实现详见代码


```
public int deleteUacMenuById(Long id, LoginAuthDto loginAuthDto) {
		Preconditions.checkArgument(id != null, "菜单id不能为空");
		int result;
		// 获取当前菜单信息
		UacMenu uacMenuQuery = new UacMenu();
		uacMenuQuery.setId(id);
		uacMenuQuery = mapper.selectOne(uacMenuQuery);
		if (PublicUtil.isEmpty(uacMenuQuery)) {
			throw new UacBizException(ErrorCodeEnum.UAC10013003, id);
		}

		...
		return result;
	}
```
### web全局异常


```
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {
	@Resource
	private TaskExecutor taskExecutor;
	@Value("${spring.profiles.active}")
	String profile;
	@Value("${spring.application.name}")
	String applicationName;
	@Resource
	private MdcExceptionLogFeignApi mdcExceptionLogFeignApi;

	/**
	 * 参数非法异常.
	 *
	 * @param e the e
	 *
	 * @return the wrapper
	 */
	@ExceptionHandler(IllegalArgumentException.class)
	@ResponseStatus(HttpStatus.OK)
	@ResponseBody
	public Wrapper illegalArgumentException(IllegalArgumentException e) {
		log.error("参数非法异常={}", e.getMessage(), e);
		return WrapMapper.wrap(ErrorCodeEnum.GL99990100.code(), e.getMessage());
	}

	/**
	 * 业务异常.
	 *
	 * @param e the e
	 *
	 * @return the wrapper
	 */
	@ExceptionHandler(BusinessException.class)
	@ResponseStatus(HttpStatus.OK)
	@ResponseBody
	public Wrapper businessException(BusinessException e) {
		log.error("业务异常={}", e.getMessage(), e);
		return WrapMapper.wrap(e.getCode() == 0 ? Wrapper.ERROR_CODE : e.getCode(), e.getMessage());
	}


	/**
	 * 全局异常.
	 *
	 * @param e the e
	 *
	 * @return the wrapper
	 */
	@ExceptionHandler(Exception.class)
	@ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
	@ResponseBody
	public Wrapper exception(Exception e) {
		log.info("保存全局异常信息 ex={}", e.getMessage(), e);
		taskExecutor.execute(() -> {
			GlobalExceptionLogDto globalExceptionLogDto = new GlobalExceptionLogDto().getGlobalExceptionLogDto(e, profile, applicationName);
			mdcExceptionLogFeignApi.saveAndSendExceptionLog(globalExceptionLogDto);
		});
		return WrapMapper.error();
	}
}
```


### Rpc全局异常



```
@Slf4j
public class Oauth2FeignErrorInterceptor implements ErrorDecoder {
	private final ErrorDecoder defaultErrorDecoder = new Default();

	/**
	 * Decode exception.
	 *
	 * @param methodKey the method key
	 * @param response  the response
	 *
	 * @return the exception
	 */
	@Override
	public Exception decode(final String methodKey, final Response response) {
		if (response.status() >= HttpStatus.BAD_REQUEST.value() && response.status() < HttpStatus.INTERNAL_SERVER_ERROR.value()) {
			return new HystrixBadRequestException("request exception wrapper");
		}

		ObjectMapper mapper = new ObjectMapper();
		try {
			HashMap map = mapper.readValue(response.body().asInputStream(), HashMap.class);
			Integer code = (Integer) map.get("code");
			String message = (String) map.get("message");
			if (code != null) {
				ErrorCodeEnum anEnum = ErrorCodeEnum.getEnum(code);
				if (anEnum != null) {
					if (anEnum == ErrorCodeEnum.GL99990100) {
						throw new IllegalArgumentException(message);
					} else {
						throw new BusinessException(anEnum);
					}
				} else {
					throw new BusinessException(ErrorCodeEnum.GL99990500, message);
				}
			}
		} catch (IOException e) {
			log.info("Failed to process response body");
		}
		return defaultErrorDecoder.decode(methodKey, response);
	}
}
```






