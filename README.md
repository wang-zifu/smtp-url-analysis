# phish-analysis-no-datastore 

[ These are newer bro policies to subtitue smtp-embedded-urls-cluster and smtp-embedded-urls-bloom.bro  ] 

Primary scope of these bro policies is to give more insights into smtp-analysis esp to track phishing events. 

This is a subset of phish-analysis repo and doesn't use any backed postgres database. So relieves the user from postgres dependency while getting basic phishing detection up and running very quickly.

Following functionality are provided by the script 

1) Works in a cluster and standalone mode 
2) extracts URLs from Emails and logs them to smtpurl_links.log 
3) Tracks these SMTP urls in http analyzer and logs if any of these SMTP URL has been clicked into a file smtp_clicked_urls.log 
4) Reads a file for malicious indicators and generates an alert of any of those inddicators have a HIT in smtp traffic (see below for more details)
5) Generates alerts if suspicious strings are seen in URL (see below for details)
6) Generates  alerts if a SMTP URL is clicked resulting in a file download 


Detail Notes: 

All the configuration variables are in the following file. Please modify as needed: 

configure-variables-in-this-file.bro 

Note: 

1) Make sure you replace site.org in the file with your domain name(s)
2) Make sure you populate Phish::smtp_indicator_feed 


Detail Alerts and descriptions: Following alerts are generated by the script: 

1) Heuristics in smtp-malicious-indicators.bro are used to flag known sensitve IoC's (sort of  your local smtp intel feed). 

This should generate following Kinds of notices:

- Malicious_MD5,
- Malicious_Attachment,
- Malicious_Indicator,
- Malicious_Mailfrom,
- Malicious_Mailto,
- Malicious_from,
- Malicious_reply_to,
- Malicious_subject,
- Malicious_rcptto,
- Malicious_path,
- Malicious_Decoded_Subject

To activate these you need to modify configure-variables-in-this-file.bro and setup path to feed file:
	ex: redef Phish::smtp_indicator_feed = "/feeds/BRO-feeds/smtp_malicious_indicators.out" ;

I have a cron job which scraps various email indicators (senders, subject, attachment, md5 hash etc) from various phish related feeds/notices and periodically creates this one fire /feeds/BRO-feeds/smtp_malicious_indicators.out. Bro reads this file using input-framework. 

New additions to file doesn't requre bro to be restarted. 

Note: 
1) Make sure the fields inthefile are <tab> seperated. 
2) Make sure format of above feed file complies to:

################### feed-file-format ####################################
#fields indicator       description
"At Your Service" <service@site.org>	Some random comment
badsender@example.com	some random comment
f402e0713127617bda852609b426caff	some bad hash
HelpDesk	some bad subject
#########################################################################

 
Example alert: 
- Phish::Malicious_rcptto

Aug 24 11:26:06 CPLZuO3KTSDHx9mCC1      174.15.3.146    36906   18.3.1.10    25      
tcp     Phish::Malicious_rcptto Malicious rectto :: [indicator=badsender@example.com, description=random test ], 
badsender@example.com	badsender@example.com	174.15.3.146 18.3.1.10	25      
bro     Notice::ACTION_EMAIL,Notice::ACTION_LOG 60.000000       F       -       -       -       -       -


2) smtp-sensitive-uris.bro will generate following alerts 

 - SensitiveURI
 - Dotted_URL
 - Suspicious_File_URL
 - Suspicious_Embedded_Text
 - WatchedFileType
 - BogusSiteURL


Example Alert: 

1503599166.565855       CPLZuO3KTSDHx9mCC1      1.1.1.1    36906   2.2.2.2    25      -       -       -       tcp     Phish::BogusSiteURL     Very similar URL to site: http://www.site.org.blah.com/ from  1.1.1.1       -       1.1.1.1    2.2.2.2  25      -       bro     Notice::ACTION_EMAIL,Notice::ACTION_LOG 3600.000000     F       -       -       -       -       -

again see configure-variables-in-this-file.bro for tweaking and tunning 



3) Malicious file download 	

	if a link in an email is clicked and results in a file download, this module can generate an alert of that as well. 

	Example alert:

1481499234.568566       CQa9SJ1adwAqlPDcKj      1.1.1.1      49067   46.43.34.31     80      FxrREO3dgcnSlAQZO8      application/x-dosexec   http://the.earth.li/~sgtatham/putty/0.67/x86/putty.exe  tcp     Phish::FileDownload     [ts=1481431889.562629, uid=CX5ROKa8g7WcfnET4, from=Bad Guy <random@gmail.com>, to=John Doe <jd@site.org>, subject=putty.exe, referrer=[]]        http://the.earth.li/~sgtatham/putty/0.67/x86/putty.exe  1.1.1.1      46.43.34.31     80      -       bro     Notice::ACTION_LOG    3600.000000     F


4) Watch for URLs which only have IP address instead of domain names in them - another sign of maliciousness
 - Phish::DottedURL 	

1483418588.406004       CNDcli3Oo5dFqrJNhi      198.124.252.166 46134   128.3.41.120    25      -       -       -       tcp     Phish::DottedURL        Embeded IP in URL http://183.81.171.242/c.jpg from  198.124.252.166     -       198.124.252.166 128.3.41.120 25       -       bro     Notice::ACTION_LOG      3600.000000     F


5) Phish::SensitiveURI

sample alert:

1351714828.429308       CAmJxI1WlO5E5bWxCj      128.3.41.133    1277    209.139.197.113 25      -       -       -       tcp     Phish::SensitiveURI     Suspicious text embeded in URL http://www.foxterciaimobiliaria.com.br/corretor/565/ from  CAmJxI1WlO5E5bWxCj -128.3.41.133    209.139.197.113 25      -       bro     Notice::ACTION_LOG      3600.000000     F


Generates an Alert when a string in URL matches signature defined in "suspicious_text_in_url" available in configure-variables-in-this-file.bro 

6) Phish::WatchedFileType - Simple regexp match on file extensions. 

[This is a noisy notice but useful for logging.  for critical files flagging use (3) above which is malicious file download based on mime-types.] 

Sample Alert: 

1481431889.683598       CxGUuzDvWCpUdFI27       74.125.83.52    35030   128.3.41.120    25      -       -       -       tcp     Phish::WatchedFileType  Suspicious filetype embeded in URL http://the.earth.li/~sgtatham/putty/0.67/x86/putty.exe from  74.125.83.52 -74.125.83.52    128.3.41.120    25      -       bro     Notice::ACTION_LOG      3600.000000     F


7) Phish::HTTPSensitivePOST is generated when a URL in an email is clicked and results in a HTTP Post request. Often this is how passwords are transmitted on phishing sites. 

Notice in alert below: username=me@me.com&tel=me&password=me 

1449085047.857802       COuvQB1n4JF3MILQUa      128.3.10.69     57106   67.227.172.217  80      -       -       -       tcp     Phish::HTTPSensitivePOST        Request: /cli/viewd0cument.dropboxxg.20gbfree.secure.verfy.l0gin.user0984987311111-config-l0gin-verfy.user763189713835763/validate.php - Data: type=G+Mail&username=me@me.com&tel=me&password=me&frmLogin:btnLogin1=&frmLogin:btnLogin1=      -       128.3.10.69     67.227.172.217  80      -       bro     Notice::ACTION_LOG      3600.000000     F




