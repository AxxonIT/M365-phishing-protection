See Write-up: https://www.linkedin.com/pulse/punishing-aitms-using-css-flask-jay-kerai-ffrhe  
 
The aim of this fork is add the ability to alert the user to not put their credentials in against AiTM/Evilgnix, this is based of the work of CIPP. Additionally I added supporting code for a teams webhook for alerting SecOps Teams. Note that this can be defeated trivially by rewriting the CSS (see https://nicolasuter.medium.com/aitm-phishing-with-azure-functions-a1530b52df05) You can however flip the logic and only get users to authenticate __ONLY__ if they see the "safe" image as the sign-in box background. This would severly hinder distributed attacks, however targeted attacks would still be possible. (Ideally this should be unique across all tenants to mitigate any CSS manipulation)

I needed a "serverless" solution for hosting for VPS and a local one so I've included both, deploy whichever you need! (gunicorn app:app)  
![image](https://github.com/jkerai1/clarion/assets/55988027/79848295-0c26-4b98-9ba7-80c3119a004e)

Server-side:
> Output below is from Render free tier

![image](https://github.com/jkerai1/clarion/assets/55988027/c9ff7224-954d-412b-9d70-bf6e9590c457)

See Defending against AiTMs/Phishing: https://www.linkedin.com/posts/jay-kerai-cyber_devfender-entra-token-activity-7122902992873287681-P03M & https://github.com/jkerai1/So-You-ve-Got-MFA-Defending-and-Responding-Against-MFA-Bypass-Techniques-in-Entra

Example of being served back "safe.png":

![image](https://github.com/jkerai1/clarion/assets/55988027/c583b3f8-8dff-4378-a3f8-594be3b63536)

CSS:
```
.ext-sign-in-box
{
    background-image: url('https://{Your Domain or IP}/companyBranding.png');
}
```

# Deploying to Azure Guide

We can use a web App to do this with the gunicorn startup command (gunicorn app:app)  
![image](https://github.com/user-attachments/assets/0e4fbd46-ddf7-45b0-88f0-bb1b15c72af1)  

I have used github auth with github action to auto deploy. I suggest forking the repo first and authorize to your own fork:  
![image](https://github.com/user-attachments/assets/183afa36-9886-41dd-ac2c-21840aedd12a)  

Then we add the web app URL (Optionally we can bind our web app to a custom domain) to our CSS to be uploaded to company branding in Entra (see CSS Above). Azure can handle the TLS/SSL for us too.    

![image](https://github.com/user-attachments/assets/e3a966d2-3ce8-4d54-9ae1-e0850ed07c84)

# Clarions original ReadMe with some edits:

# clarion
The clarion call tells you if someone is logging into an AitM proxy that is proxying your company's M365 login page

![image](https://github.com/HuskyHacks/clarion/assets/57866415/58627a15-8beb-43d5-a0f1-4172f9da8653)

## Warning
This is **extremely** experimental and can be defeated:

![image](https://github.com/user-attachments/assets/2f3c345a-0bb8-4946-b8c1-e31390e0c0ca)  

![image](https://github.com/user-attachments/assets/5dbdbfcd-e366-429a-9b65-155f6897bf1a)  

![image](https://github.com/user-attachments/assets/a4ea4c5e-5387-471b-a0ed-37c0d9977640)  

Here is a good example of swapping referrer to appear legitimate https://github.com/rvdwegen/vdwegen-api/blob/main/tenantDetails/run.ps1#L106  

Also See: https://insights.spotit.be/2024/06/03/clipping-the-canarys-wings-bypassing-aitm-phishing-detections/  

## Concept & Disclaimer
[This article](https://zolder.io/using-honeytokens-to-detect-aitm-phishing-attacks-on-your-microsoft-365-tenant/) from Zolder describes the concept quite well. This is not my original idea and the credit goes to them for it.

M365 allows you to inject custom CSS into the M365 login screen through the Company Branding settings. Ostensibly, this allows you to put a cool background image on your login page that fits your company branding.

We can take advantage of this to detect when a user is logging into an Adversary in the Middle (AitM) proxy like Evilginx that is mimicking your legitimate login page.

Clarion creates and hosts a small tracking pixel that we can embed into our custom company branding CSS file. When a normal login occurs, this CSS is retrieved dynamically and rendered on the M365 login page. When that happens during a routine login, the referer header for the CSS retrieval is `login.microsoftonline.com`

However, if a threat actor has created an AitM domain and login page and someone puts their username in to log in, the CSS will get pulled by the transparent proxy and the referer header will be the AitM proxy domain. If our tracking pixel is ever requested with a referer header that is NOT `login.microsoftonline.com`, it is highly likely that someone is logging into a transparent proxy (DISCLAIMER: MAJOR ASSUMPTIONS WITH THIS CONCEPT. DO YOUR OWN TESTING TO MAKE SURE THIS IS THE CASE!).

### So, why Clarion?
Zolder uses their own site, [didsomeoneclone.me](https://didsomeoneclone.me/) as their proof of concept. It works like a charm! But I wanted to create the whole system to learn more about it and demonstrate the entire process, start to finish. Thank you to Zolder for their work on this! Really cool concept and great work.

Additionally, I'm sure there are people out there that would love to use this detection capability but want to host it on their own infrastructure for privacy reasons. Clarion demonstrates a very simple, naive approach to how one would implement the tech required to alert on AitM attacks in progress.
 
## Setup
For testing, you can use a self-signed certificate. 

```
openssl req -x509 -newkey rsa:4096 -nodes -out cert.pem -keyout key.pem -days 365
```

**Important:** Make sure you accept the warning about the invalid certificate in the browser that you want to test if using a self-signed cert. If you don't, the client browser will throw an error when trying to load the remote CSS and will not trigger the clarion call.

![image](https://github.com/HuskyHacks/clarion/assets/57866415/2d66f9aa-09c5-4d15-974e-3f71e2ddcf50)

For production use, you would want a legitimate signed certificate.

Clone the repo to your publically accessible host and install the requirements:

```
$ pip3 install -r requirements.txt
```

Then run the app (you need root level permissions to bind to port 443):

```
$ [sudo] python3 app.py

 * Serving Flask app 'app'
 * Debug mode: on
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on all addresses (0.0.0.0)
 * Running on https://127.0.0.1:443
 * Running on https://[Public IP]:443
Press CTRL+C to quit
 * Restarting with stat

[*] Your public IP address is: [Public IP]
[*] Embed this pixel in your CSS file with the following code:

[some CSS element] {
    background-image: url('https://[Public IP]/[pixel name].png');
    background-size: 0 0;
}

 * Debugger is active!
 * Debugger PIN: 422-205-484
```

Take this custom CSS pixel element and add it to your M365 Company Branding as a custom CSS file:

![image](https://github.com/HuskyHacks/clarion/assets/57866415/c94192ed-6b73-43ea-a158-ca34b69f91e2)

![image](https://github.com/HuskyHacks/clarion/assets/57866415/bc24d94b-14d4-4dde-a668-09bf8482d811)

![image](https://github.com/HuskyHacks/clarion/assets/57866415/f3401d09-1bfc-4154-9971-59b07523d44e)

![image](https://github.com/HuskyHacks/clarion/assets/57866415/f9234950-9353-41d1-8b44-9b23abe69095)

The [M365 Company Branding CSS Schema](https://learn.microsoft.com/en-us/entra/fundamentals/reference-company-branding-css-template) has the list of CSS elements that the M365 login page can use. `.ext-footer` seems to be a good option to trigger the clarion call.

eg. 
```
.ext-footer {
    background-image: url(https://[Public IP]/[pixel name].png);
    background-size: 0 0;
}
```

## Future Work
I'm willing to bet that this works with login pages for any service that allows you to specify custom CSS. I don't know which ones do, so if you have any ideas, open a PR or issue and let me know!

## Ref & Acknowledgements
Thank you again to Zolder for the original article on this technique!

- https://zolder.io/using-honeytokens-to-detect-aitm-phishing-attacks-on-your-microsoft-365-tenant/

