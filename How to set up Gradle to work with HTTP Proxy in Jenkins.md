If your Jenkins server is behind a proxy, set up is required for Gradle to download dependencies from internet. 

Assume the Jenkin service s is running as local system account. Local system account cannot authenticate with proxy server.

Solution:
=========
1.	Remote login to the server using your own account. Create gradle.properties in .gradle folder of your home directory. Put in your credentials. Refer to: http://stackoverflow.com/questions/8938994/gradlew-behind-a-proxy. Take note that you must use "\\\\" or "/" to specify Windows domain users. e.g. "DOMAIN\\\\USER" or "DOMAIN/USER". Or, use systemProp.https.auth.ntlm.domain to specify domain name separately. Refer to: https://docs.gradle.org/current/userguide/build_environment.html#sec:accessing_the_web_via_a_proxy
2.	Change the security permission of the file to be visible to only your own account and Local System account to protect your password. Remove access by admin.
3.	Set or change the system environment variable GRADLE_USER_HOME to the .gradle folder in your home directory. This will make gradle use that folder as its user home directory and use the gradle.properties file and cache folder inside it. This gradle.properties file will supersede the one in project folder, if any.
4.	Restart Jenkins service after you change GRADLE_USER_HOME.
In this way, each developer can create his own gradle.properties file in the server that is only visible to himself. This means he can run gradle manually in the server if he need. At the same time, each one has the freedom to repoint GRADLE_USER_HOME to his own gradle.properties file if the current one is not working (e.g. if a developer has already left the team and his password has expired).
