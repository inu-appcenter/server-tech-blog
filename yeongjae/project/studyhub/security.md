2023 11월 22일 앱센터 15기 서버파트 스터디를 진행중, 앱센터 내 도서관리 시스템의 아키텍쳐를 구축하고 있었습니다.

앱센터 회원이 학교 이메일로 인증을 받아 로그인을 진행할 때, 스프링 시큐리티를 사용해 필터 단에서 처리해줄 지 직접 DB와 통신해 로그인을 처리할지 의논하던 중 불현듯 한가지 생각이 들었습니다.

StudyHub에서 시큐리티를 왜 사용하고있지??....

프로젝트를 처음 진행할 때를 생각해보면 정말 무지성으로 "로그인 하는데 스프링 시큐리티를 안써?? 무조건 써야지!" 라는 안일한 생각으로 시큐리티를 사용했었습니다.

이유없이 무지성으로 작성한 코드(기술)는 유지보수의 후폭풍을 몰고온다는 사실을 StudyHub 프로젝트에서 체감했기 때문에 StudyHub 프로젝트에서 시큐리티를 썼을때의 장단점을 확실하게 정리한 뒤 사용 여부를 결정해야겠다 생각했습니다.

---

---

``` java
@Configuration
@RequiredArgsConstructor
public class SecurityConfig {

    private final CorsConfig corsConfig;
    private final JwtAuthenticationFilter jwtAuthenticationFilter;
    private final JwtAuthorizationFilter jwtAuthorizationFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
                .csrf().disable()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .formLogin().disable()
                .httpBasic().disable()
                .apply(new MyCustomDsl())
                .and()
                .authorizeRequests()
                .antMatchers("/api/users/login").permitAll()
                .antMatchers("/api/users/signup").permitAll()
                .and().build();
    }
    
    public class MyCustomDsl extends AbstractHttpConfigurer<MyCustomDsl, HttpSecurity> {
        @Override
        public void configure(HttpSecurity http) {
            http
                    .addFilter(corsConfig.corsFilter())
                    .addFilter(jwtAuthenticationFilter)
                    .addFilter(jwtAuthorizationFilter);
        }
    }

}
```
<br></br>
프로젝트의 필터는 corsFilter -> jwtAuthenticationFilter -> jwtAuthorizationFilter 의 순서로 구성되어있습니다.
<br></br>
``` java
@Slf4j
@Component
public class JwtAuthenticationFilter extends UsernamePasswordAuthenticationFilter {

    private final AuthenticationManager authenticationManager;
    private final JwtProvider jwtProvider;
    private SignUpRequest signUpRequest = new SignUpRequest();

    @Value("${jwt.secret}")
    private String SECRET;

    public JwtAuthenticationFilter(AuthenticationManager authenticationManager, JwtProvider jwtProvider) {
        super(authenticationManager);
        this.authenticationManager = authenticationManager;
        this.jwtProvider = jwtProvider;
        setFilterProcessesUrl("/api/users/login");
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
        ObjectMapper om = new ObjectMapper();

        try {
            signUpRequest = om.readValue(request.getInputStream(), SignUpRequest.class);
        } catch(Exception e) {
            e.printStackTrace();
        }

        UsernamePasswordAuthenticationToken authenticationToken =
                new UsernamePasswordAuthenticationToken(signUpRequest.getEmail(), signUpRequest.getPassword());

        return authenticationManager.authenticate(authenticationToken);
    }

    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain, Authentication authResult){
        PrincipalDetails principalDetails = (PrincipalDetails) authResult.getPrincipal();
        String accessToken = jwtProvider.accessTokenCreate(principalDetails.getUser().getId());
        String refreshToken = jwtProvider.refreshTokenCreate(principalDetails.getUser().getId());

        response.addHeader(JwtProperties.ACCESS_HEADER_STRING, JwtProperties.TOKEN_PREFIX + accessToken);
        response.addHeader(JwtProperties.REFRESH_HEADER_STRING, JwtProperties.TOKEN_PREFIX + refreshToken);

        SignUpInfo signUpInfo = new SignUpInfo(signUpRequest, accessToken, refreshToken);

        CustomResponseUtil.success(response, signUpInfo);
    }
}
```
<br>
JwtAuthenticationFilter 에선 /api/users/login 으로 들어오는 경로에 대해 attemptAuthentication 메소드를 수행합니다.

attemptAuthentication 메소드는 requestBody 정보를 역직렬화 해 signUpReqeust 정보로 만든 뒤 SecurityContextHolder에 저장합니다.

SecurityContextHolder에 저장된 Authentication 객체는 추후 사용자 로그인 여부 판단에 사용되게 됩니다.
<br></br>

``` java
@Component
public class JwtAuthorizationFilter extends BasicAuthenticationFilter {

    private final JwtProvider jwtProvider;

    @Value("${jwt.secret}")
    private String SECRET;

    public JwtAuthorizationFilter(AuthenticationManager authenticationManager, JwtProvider jwtProvider) {
        super(authenticationManager);
        this.jwtProvider = jwtProvider;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        if(isHeaderVerify(request)) {
            String accessToken = request.getHeader(JwtProperties.ACCESS_HEADER_STRING);

            try {
                PrincipalDetails principalDetails = jwtProvider.accessTokenVerify(accessToken);

                Authentication authentication = new UsernamePasswordAuthenticationToken(principalDetails, null, null);

                SecurityContextHolder.getContext().setAuthentication(authentication);
            } catch(TokenExpiredException e) {
                throw new TokenNotFoundException();
            }
        }

        chain.doFilter(request, response);
    }

    private boolean isHeaderVerify(HttpServletRequest request) {
        String accessHeader = request.getHeader(JwtProperties.ACCESS_HEADER_STRING);

        if(accessHeader == null) {
            return false;
        }
        return true;
    }
}
```
<br>
JwtAuthorizationFilter에선 토큰이 존재할 경우 이를 Authentication 객체로 만들어 SecurityContextHolder에 저장하고 있습니다.

이 객체엔 단점이 두가지 있습니다.

1. AuthenticationFilter에서 SecurityContextHolder에 저장된 Authentication 객체가 사용자 로그인 여부에 판단되고있지 않습니다.
2. AuthorizationFilter는 쓰이지도 않는데 모든 API 호출에 대해 Filter로 작용하고 있기 때문에 성능상 불이익이 있습니다.

<br>AuthorizationFilter 에선 Jwt를 이용해 새로운 Authentication 객체를 만들어 SecurityContextHolder에 다시 저장만 하고있기 때문에 로그인 여부 판단과는 전혀 관계없는 일을 하고 있는 것 입니다. 결국 처음 의도했던 SecurityContextHolder에서 값을 가져오는 행동을 전혀 하고 있지 않는다는 것 입니다.
그렇다면 지금까지 로그인 여부 판단은 어디서 했냐??

아래 코드인 UserIdArgumentResolver에서 진행하고 있었습니다.

``` java
@Component
@RequiredArgsConstructor
public class UserIdArgumentResolver implements HandlerMethodArgumentResolver {

    @Value("${jwt.secret}")
    private String SECRET;

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.getParameterType() == UserId.class;
    }

    @Override
    public UserId resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
                                                    NativeWebRequest webRequest, WebDataBinderFactory binderFactory) {
        try {
            String jwtToken = webRequest.getHeader(JwtProperties.ACCESS_HEADER_STRING);
            if (jwtToken == null) {
                return new UserId(null);
            }
            jwtToken = jwtToken.replace(JwtProperties.TOKEN_PREFIX, "");
            Long id = (JWT.require(Algorithm.HMAC512(SECRET)).build().verify(jwtToken).getClaim("id")).asLong();
            return new UserId(id);
        } catch (Exception e) {
            return null;
        }
    }
}
```

UserIdArgumentResolver에서 로그인 여부를 판별해주는데 아무 일도 안하는 AuthorizationFilter는 있을 필요가 없다 생각했습니다.

또한 JwtAuthenticationFilter에서 진행중인 로그인의 경우도 Service 단에서 직접 진행할 수 있습니다.

Spring security 의존성을 가지고 있으면 테스트 코드 작성도 까다롭다는 단점 또한 있어 Spring security 의존성을 삭제하기로 결정했습니다.

하지만 의존성을 삭제할 때 문제가 한가지 있었습니다.

사용자 회원가입 시 비밀번호를 인코딩해 DB에 저장해야하는데 인코딩을 하는 객체가 시큐리티 라이브러리 안에 있는 것 이었습니다. 이를 해결하기 위해 Spring Security 공식문서를 통해 패스워드 인코더 객체에 대해 학습 한 뒤 인코딩을 수행하는 객체를 직접 만들기로 결정했습니다.

<br>

![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/3a6193cf-82a9-4c5b-845b-9af8c147b858)


<br>
스프링 시큐리티의 패스워드 인코더가 양방향 암호화가 아닌 단방향 암호화로 작동된다고 나와있습니다.


단방향 암호화로 작동되면 비밀번호 검증 시 두가지 문제가 있었습니다.


1. 사용자가 입력한 비밀번호를 통해 DB 내 비밀번호와의 일치 여부를 판단 문제.

2. 비밀번호가 중복될 시 해시값 충돌 문제.


1번의 해결방법은 간단합니다. 사용자의 비밀번호 값을 DB에 암호화 해 저장했으니 검증을 위해 입력한 비밀번호 값 또한 암호화 한 뒤 대조하면 됩니다.


StudyHub의 사용자들의 이메일은 각자 다르기 때문에 인코딩 이메일을 솔트값으로 넣어줘 2번 문제를 해결했습니다.


구글링해 찾아보니 이미 단방향 암호화로 패스워드 인코더를 멋지게 구현해놓은 분이 계셔 참고해 객체를 완성했습니다.

``` java
@Component
public class PasswordEncoder {
​
    public String encode(String email, String password) {
        try {
            KeySpec spec = new PBEKeySpec(password.toCharArray(), getSalt(email), 85319, 128);
            SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA1");
​
            byte[] hash = factory.generateSecret(spec).getEncoded();
            return Base64.getEncoder().encodeToString(hash);
        } catch (NoSuchAlgorithmException | UnsupportedEncodingException | InvalidKeySpecException e) {
            throw new RuntimeException(e);
        }
    }
​
    public boolean matches(String password, String decodedPassword) {
        return password.equals(decodedPassword);
    }
​
    private byte[] getSalt(String email) throws NoSuchAlgorithmException, UnsupportedEncodingException {
        MessageDigest digest = MessageDigest.getInstance("SHA-512");
        byte[] keyBytes = email.getBytes(StandardCharsets.UTF_8);
​
        return digest.digest(keyBytes);
    }
}
```

<br>
이제 이 인코더 객체를 서비스 로직에서 호출 해 인코딩한 문자열로 변환시킨 뒤 DB에 저장하는 방식으로 회원가입을 진행 했습니다.

---

### 결론

설계할때 기능이 왜 필요한지 생각하고 설계하자!


**references**

https://github.com/study-hub-inu/study-hub-server

https://spring.io/projects/spring-security

https://wonchan.tistory.com/4