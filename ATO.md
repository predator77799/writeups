
<script>
window.onload = function() {

var chart = new CanvasJS.Chart("chartContainer", {
	animationEnabled: true,
	title: {
		text: "Publicly disclosed Account Takeover reports on HackerOne"
	},
	data: [{
		type: "pie",
		startAngle: 240,
		indexLabel: "{label} {y}",
		dataPoints: [
			{y: 16, label: "Bugs in OAuth flow"},
			{y: 15, label: "CSRF"},
			{y: 14, label: "OTP/Password/Password reset token brute force"},
			{y: 8, label: "No restrictions while changing email/password/phone number"},
			{y: 5, label: "XSS"},
			{y: 4, label: "Improper session management"},
			{y: 11, label: "Miscellaneous"}
		]
	}]
});
chart.render();

}
</script>
Hey guys! This is Kuldeep from 1daylabs!

Today's writeup is about an analysis of publicly disclosed account takeover reports of hackerone. In this writeup there will just be an analysis and in upcoming write-ups, I will be showing how each one of them are exploited. These write-ups will be impact focused. I decided to make write-ups for this is because many people including me stumble upon a bug and do not realize the criticality or exploitability of it.

You can find account takeover reports with this Google dork `inurl:hackerone.com/reports "Account Takeover"` but you may find duplicate search results in Google so I just searched for "Account Takeover" in [Hackerone Hacktivity](https://hackerone.com/hacktivity). However, not all of them were account takeovers. The search results included subdomain takeovers as well. After analyzing 186 reports, I found 73 valid account takeovers. Overview of those 73 reports is as follows:

<br><div id="chartContainer" style="height: 300px; width: 100%;"></div>
<script src="https://canvasjs.com/assets/script/canvasjs.min.js"></script><br>

###Bugs in OAuth flow

One of the most common issue during authentication is bug in OAuth flow. Bugs affecting the OAuth flow were:

1.	`access_token` not validated on server side
2.	Open redirect using `redirect_uri`
3.	`code` parameter not handled properly
4.	And other complex OAuth bugs

I have not summarized every single scenario because some bugs are too complex to be summarized in one line.

We will come up with a new blog explaining OAuth bugs and explaining reports but till then you can read [this awesome article explaining OAuth and OAuth bugs](https://medium.com/a-bugz-life/the-wondeful-world-of-oauth-bug-bounty-edition-af3073b354c1).

###CSRF

Next widely found issue leading to account takeover is CSRF. CSRF bugs seem too simple but they can have serious consequences which can result in account takeover. Common CSRF bugs I found were:

1.	CSRF in change email address functionality
2.	CSRF in mobile number change functionality

An attacker can send you an innocent looking HTML page which may have evil content which exploits a CSRF vulnerability to change your email address/phone number. After changing your email address/phone number, attacker can easily exploit forgot password functionality to reset your password and take over your account.

###OTP/Password/Password reset token brute force

The next issue which is often seen is lack of rate limiting protections on sensitive functions such as One-Time-Password(OTP) submit, password submit and password reset. A rate limiting mechanism blocks access to a resource after certain amount of requests. For example, when you request for OTP and you have to wait for certain amount of time(e.g 2 minutes) to be able to request OTP again. The same applies to OTP submit and password submit mechanisms as well. Another example, if your OTP is 4 characters long and no rate limiting mechanisms are used then an attacker can brute force the OTP. It would be cracked within few seconds to few minutes. I created the following PHP script to test brute forcing on my local machine.

```
<?php
if($_GET['otp'] == 9999) {
        echo "voila!";
} else {
        header('HTTP/1.0 403 Forbidden');
}
?>
```

And then brute forcing the OTP with [ffuf](https://github.com/ffuf/ffuf):


```
➜	ffuf -u http://127.0.0.1/cracktest.php\?otp\=FUZZ -w pins.txt -mc 200

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.1.0-git
________________________________________________

 :: Method           : GET
 :: URL              : http://127.0.0.1/cracktest.php?otp=FUZZ
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200
________________________________________________

9999                    [Status: 200, Size: 6, Words: 1, Lines: 1]
:: Progress: [10000/10000] :: Job [1/1] :: 5000 req/sec :: Duration: [0:00:02] :: Errors: 0 ::
```
And the OTP 9999 got brute forced in 2 seconds. However, this is obvious that this will take less time as the script is in my local machine. Although this will take a little slower on a remote website depending on your internet speed and available system resources.

###No restrictions while changing email/password/phone number

Next issue is no restrictions while changing email address/password/phone number. Some developers may think that after authentication, user must be allowed to perform all functions without requiring re-authentication. But when developers do not make re-authentication mandatory on sensitive functions like changing email address/password/phone number, it becomes an issue.

Suppose, you are working on a shared system and multiple people accesses a computer. For example, in a cyber cafe, in a library, in a school etc. If you forget to log out on a shared system, someone else having malicious intents can change the email address or phone number or even password and then he has access to the account. For example, you log into your account on your school PC. And a bad person comes and changes your email without you knowing. And then he resets the password by using forgot password functionality. Thus, taking over your account!

This can be easily fixed by requiring password or other sort of authentication mechanism in sensitive functions like changing email, changing password and changing mobile number.

You may think that this kind of vulnerability requires physical access but some programs do accept this kind of bugs. This is why this is on 4th position.

###XSS

Cross side scripting makes it way to 5th position in this list. There are a lot of XSS reports but not all of them leads to account takeover. An XSS only becomes cause of account takeover if cookies are stored insecurely which is without `HttpOnly` flag.

`HttpOnly` flag tells browser that the cookie must not be allowed to access by Javascript/VBScript and it must be only sent to server over HTTP.

Due to lack of `HttpOnly` flag in session cookies, it is possible for an attacker having an XSS, to takeover account by stealing cookies.

So, next time you find an XSS, try leveraging it to account takeover. It may increase your payout.


###Improper Session Management

This issue arises when websites do not expire sessions after a user has changed password/email address/phone number. Websites should expire all active sessions after changing password/email address/ mobile number. Or if not expired automatically, at least ask to review active sessions.

This vulnerability can be exploited when user has already got his account hacked by any means and he is trying to change his password or email address or phone number. Even after he changes his password, attacker is still logged in to the account. Meaning he can still make changes to the account even after victim has changed the password or email address.

###Miscellaneous

This class includes different unique vulnerabilities which cannot be categorized. This vulnerabilities are:

1.	Password reset link not expiring
2.	HTTPS not used during transmission of sensitive data
3.	No validation of password reset token
4.	API revealing password reset hash value
5.	Bug in differentiating SSO account and normal account
6.	Host header injection
7.	Open redirection in login page
8.	Subdomain Takeover
9.	Web cache deception
