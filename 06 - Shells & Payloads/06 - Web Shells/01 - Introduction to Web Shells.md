# Introduction to Web Shells

Almost entirely context. One actionable mental model:

Web shell = initial foothold, not a stable working environment. Many apps auto-delete uploads. Use the web shell to immediately upgrade to a reverse shell — don't do post-ex work through the browser.

Common upload vectors to check on every web app engagement: unrestricted file upload forms, profile picture/avatar upload fields (bypass client-side checks), Tomcat/Axis2/WebLogic WAR deployment, misconfigured FTP with write access to webroot, authenticated app functionality.

Payload language must match the server-side stack — PHP for Apache/Nginx/PHP, JSP for Tomcat/Axis2/WebLogic, ASPX for IIS. Upload the wrong type and it renders as plaintext, nothing executes.