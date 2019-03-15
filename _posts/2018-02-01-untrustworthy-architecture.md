# Untrustworthy Architecture

The architecture built when you can’t trust other systems your project is consuming.

When we started to write our website we start to talk about what rules, what patterns and what architecture we should follow because we have some ideas on what to do but not all our knowledge was, as usual, short.
We want to “Keep it simple” because the simple code is easy to maintain and easy to refactor, but soon we discovered will not be so easy.

## Queue on Queue

In our project, we need to implement a simple task such as send email to our customers (for password reset or simple notifications) and keep track of which emails are sent and when. The first option was: “ok just send emails directly to our relay server and log them after send to keep track”, it’s easy just call SmtpClient.SendMail from our code and the work is done right?.
Yes but not in this case because we need to handle the error-prone system we have to use. For this problem, we should build a Queue and send emails from that Queue to the relay server and manage Errors, retry and so on. The Architect in our team said but you are building a Queue on Queue and it’s a smell, my answer was yes I know but in this case is the only way I can trust our system.

## The reason?

The reason is I experienced a lot of problem in troubleshooting when some customer has email related problems and I need in some cases to be sure our system is working as expected and have proof of that and I can demonstrate it. I was not sure but the day we had to investigate on some strange email received by our customers we discovered the only system that logs something is our and we discovered there was a Linux SMTP server “script” that sometimes under heavy load mess the email content and send to customers, from all other systems there is no trace because they don’t log anything.

## Then what is the lesson?

The lesson is we can consider my implementation an over-engineering architecture for send just some email and in most case this is true, but in some case when you can’t trust the system you use, you need to complicate your architecture without any technical reasons. 
I think is not over-engineered but it is just an Untrustworthy Architecture the best name for this kind of solution. The solution you need to implement when you can’t control another system you are consuming or simply you don’t trust it.