---
layout: post
title:  "Sending Emails Using Java"
date:   2018-05-24 12:18:00 +0200
categories: java email testing
---
When sending an email using Java code is required, good old SMTP and javax.mail package do all the work. What's left for developer if properly align and pass all required parameters.

Good start to read about Java Mail API is probably [here][javamail-oracle]. When it comest to code, is makes sense to go directly to [github][javamail-github]. Usage is pretty straightforward and well described, so I won't stop here (good exampe is [here][aws-smtp-java]). Where I will stop, is how to actually introduce test coverage for email integration code.

It appears that answer to the question "how do I test java email client code" has already been given back in 2011 on [stackoverflow][wiser-answer]. This answer leads, surprise-surprise, to [github][smtp-github], where basic usage of email server code is [described][smtp-server]. And finally, regarding testing, the answer is [given][smtp-wiser] as well.

Adding javax.mail and wiser dependencies sound like a good start, so here how you do it with gradle:

```
compile('com.sun.mail:javax.mail:1.6.1')
testCompile('org.subethamail:subethasmtp-wiser:1.2')
```

Now some code how to send an email with zipper attachment. First required properties are put into `Properties` instance.

```java
Properties properties = new Properties();
properties.put("mail.smtp.host", "localhost");
properties.put("mail.smtp.user", "localuser");
properties.put("mail.smtp.password", "password");
properties.put("mail.smtp.port", 2500);
properties.put("mail.smtp.auth", false);
```

Next session and message are being built.

```java
String recipient = "user@example.com";
String sender = "noreply@example.com";
String subject = "Hello there";
String content = "Hello there";

String payload = "{\"isJson\":true}";
String filename = "data.json";

Session session = Session.getDefaultInstance(properties);
MimeMessage mimeMessage = new MimeMessage(session);
try {
    InternetAddress address = new InternetAddress(recipient);
    mimeMessage.addRecipient(Message.RecipientType.TO, address);

    BodyPart message = new MimeBodyPart();
    message.setText(content);

    Multipart multipart = new MimeMultipart();
    multipart.addBodyPart(message);

    BodyPart attachment = new MimeBodyPart();
    DataSource dataSource = new ByteArrayDataSource(Utils.zip(payload, filename), "application/zip");
    attachment.setDataHandler(new DataHandler(dataSource));
    attachment.setFileName(filename + ".zip");
    multipart.addBodyPart(attachment);

    mimeMessage.setContent(multipart);
    mimeMessage.setSubject(subject);
    mimeMessage.setSender(new InternetAddress(sender));
    mimeMessage.setFrom(new InternetAddress(sender));

    Transport.send(mimeMessage);
} catch (MessagingException | IOException e) {
    logger.error("Failed to send email", e);
}
```

Now we can write a unit-test using Wiser.

```java
Wiser wiser = new Wiser();
wiser.setPort(2500);
wiser.start();

// Now send some email
Email.send(
	"inbox@example.com",	// recipient
	"noreply@example.com",	// sender
	"Hello there",			// subject
	"See attachment", 		// content
	"{\"attached\":true}", 	// payload of the attachment
	"payload.json"			// file name for the attachment
);

List<WiserMessage> messages = wiser.getMessages();
assertEquals(1, messages.size());

assertEquals("inbox@example.com", messages.get(0).getEnvelopeReceiver());
assertEquals("Hello there", messages.get(0).getMimeMessage().getSubject());

MimeMultipart multipart = (MimeMultipart) messages.get(0).getMimeMessage().getContent();
assertEquals("See attachment", multipart.getBodyPart(0).getContent());
assertEquals("payload.json.zip", multipart.getBodyPart(1).getFileName());

Map<String, String> data = unzip(multipart.getBodyPart(1).getInputStream());
assertEquals(1, data.size());
assertTrue(data.containsKey("payload.json"));
assertEquals("{\"attached\":true}", data.get("payload.json"));

wiser.stop();
```

And finally the code to zip/unzip the data, really straightforward.

```java
public static byte[] zip(String payload, String fileName) throws IOException {
    byte[] bytes = payload.getBytes();
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    ZipOutputStream zos = new ZipOutputStream(baos);
    ZipEntry entry = new ZipEntry(fileName);
    entry.setSize(bytes.length);
    zos.putNextEntry(entry);
    zos.write(bytes);
    zos.close();
    return baos.toByteArray();
}

public static Map<String, String> unzip(InputStream is) throws IOException {
    Map<String, String> result = new HashMap<>();
    ZipInputStream zis = new ZipInputStream(is);
    byte[] buffer = new byte[1024];
    ZipEntry entry = zis.getNextEntry();
    while (entry != null) {
        String name = entry.getName();
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        int len;
        while ((len = zis.read(buffer)) > 0) {
            baos.write(buffer, 0, len);
        }
        baos.close();
        String data = new String(baos.toByteArray());
        result.put(name, data);
        entry = zis.getNextEntry();
    }
    return result;
}
```

That's all folks! Now you see how easy it is to introduce unit-tests for the code that sends emails. Receiving emails is a different story though and won't be covered here.

[javamail-oracle]: http://www.oracle.com/technetwork/java/javamail/index.html
[javamail-github]: https://javaee.github.io/javamail/
[aws-smtp-java]: https://docs.aws.amazon.com/ses/latest/DeveloperGuide/send-using-smtp-java.html
[wiser-answer]: https://stackoverflow.com/a/8604585
[smtp-github]: https://github.com/voodoodyne/subethasmtp
[smtp-server]: https://github.com/voodoodyne/subethasmtp/blob/master/UsingSubEthaSMTP.md
[smtp-wiser]: https://github.com/voodoodyne/subethasmtp/blob/master/Wiser.md
