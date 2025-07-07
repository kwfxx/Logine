import requests, uuid, random, re, json, hashlib, string, os, time
from time import sleep

# ثابت
uu = '83f2000a-4b95-4811-bc8d-0f3539ef07cf'
timestamp = str(int(time.time()))

# دوال توليد عشوائية
def RandomString(n=10):
    letters = string.ascii_lowercase + '1234567890'
    return ''.join(random.choice(letters) for _ in range(n))

def RandomStringChars(n=1):
    letters = string.ascii_lowercase
    return ''.join(random.choice(letters) for _ in range(n))

def randomStringWithChar(stringLength=10):
    letters = string.ascii_lowercase + '1234567890'
    result = ''.join(random.choice(letters) for _ in range(stringLength - 1))
    return RandomStringChars(1) + result

# توليد user-agent عشوائي
def generateUSER_AGENT():
    Devices_menu = ['HUAWEI', 'Xiaomi', 'samsung', 'OnePlus']
    DPIs = ['480', '320', '640', '515', '120', '160', '240', '800']
    randResolution = random.randrange(2, 9) * 180
    lowerResolution = randResolution - 180
    DEVICE_SETTINTS = {
        'system': "Android",
        'Host': "Instagram",
        'manufacturer': random.choice(Devices_menu),
        'model': f"{random.choice(Devices_menu)}-{randomStringWithChar(4).upper()}",
        'android_version': random.randint(18, 25),
        'android_release': f"{random.randint(1, 7)}.{random.randint(0, 7)}",
        "cpu": f"{RandomStringChars(2)}{random.randrange(1000, 9999)}",
        'resolution': f'{randResolution}x{lowerResolution}',
        'randomL': RandomString(6),
        'dpi': random.choice(DPIs)
    }
    return '{Host} 155.0.0.37.107 {system} ({android_version}/{android_release}; {dpi}dpi; {resolution}; {manufacturer}; {model}; {cpu}; {randomL}; en_US)'.format(**DEVICE_SETTINTS)

# توليد Device ID
def generate_DeviceId(ID):
    volatile_ID = "12345"
    m = hashlib.md5()
    m.update(ID.encode('utf-8') + volatile_ID.encode('utf-8'))
    return 'android-' + m.hexdigest()[:16]

# رؤوس الطلب
def headers_login(user_agent):
    return {
        'User-Agent': user_agent,
        'Host': 'i.instagram.com',
        'content-type': 'application/x-www-form-urlencoded; charset=UTF-8',
        'accept-encoding': 'gzip, deflate',
        'x-fb-http-engine': 'Liger',
        'Connection': 'close'
    }

# التحقق الثنائي
def checkpoint(req, cookies, user_agent, username, csrftoken):
    info = requests.get(f"https://i.instagram.com/api/v1{req.json()['challenge']['api_path']}", headers=headers_login(user_agent), cookies=cookies)
    step_data = info.json()["step_data"]

    if "phone_number" in step_data:
        print(f"[0] phone_number: {step_data['phone_number']}")
    elif "email" in step_data:
        print(f"[1] email: {step_data['email']}")
    else:
        print("unknown verification method")
        input()
        exit()
    
    return send_choice(req, cookies, user_agent, username, csrftoken)

def send_choice(req, cookies, user_agent, username, csrftoken):
    choice = input('choice : ')
    data = {
        'choice': str(choice),
        '_uuid': uu,
        '_uid': uu,
        '_csrftoken': csrftoken
    }
    challnge = req.json()['challenge']['api_path']
    send = requests.post(f"https://i.instagram.com/api/v1{challnge}", headers=headers_login(user_agent), data=data, cookies=cookies)
    contact_point = send.json()["step_data"]["contact_point"]
    print(f'code sent to: {contact_point}')
    return get_code(req, cookies, user_agent, username, csrftoken)

def get_code(req, cookies, user_agent, username, csrftoken):
    code = input("code : ")
    data = {
        'security_code': code,
        '_uuid': uu,
        '_uid': uu,
        '_csrftoken': csrftoken
    }
    path = req.json()['challenge']['api_path']
    send_code = requests.post(f"https://i.instagram.com/api/v1{path}", headers=headers_login(user_agent), data=data, cookies=cookies)

    if "logged_in_user" in send_code.text:
        print(f'Login Successfully as @{username}')
        sessionid = send_code.cookies.get("sessionid")
        csrftoken = send_code.cookies.get("csrftoken")
        print(f"session : {sessionid}")
        print(csrftoken)
        return sessionid
    else:
        try:
            regx_error = re.search(r'"message":"(.*?)",', send_code.text).group(1)
            print(regx_error)
        except:
            print("Invalid or unexpected response.")
        ask = input("Code not valid. Try again? [Y/N]: ")
        if ask.lower() == "y":
            return get_code(req, cookies, user_agent, username, csrftoken)
        else:
            exit()

# تسجيل الدخول
def Login():
    username = input('UserName? : ')
    password = input('Password? : ')
    device_id = generate_DeviceId(username)
    user_agent = generateUSER_AGENT()

    data = {
        'guid': uu,
        'enc_password': f"#PWD_INSTAGRAM:0:{timestamp}:{password}",
        'username': username,
        'device_id': device_id,
        'login_attempt_count': '0'
    }

    req = requests.post("https://i.instagram.com/api/v1/accounts/login/", headers=headers_login(user_agent), data=data)
    cookies = req.cookies
    csrftoken = cookies.get("csrftoken")

    if not csrftoken:
        # لو التوكن غير موجود نحط قيمة افتراضية أو نطبع تحذير
        print("[!] csrftoken not found in cookies, using fallback value.")
        csrftoken = "missing"

    if "logged_in_user" in req.text:
        print(f'Login Successfully as @{username}')
        sessionid = cookies.get("sessionid")
        print(f"session : {sessionid}")
        print(f"csrftoken : {csrftoken}")
        return sessionid
    elif 'checkpoint_challenge_required' in req.text:
        print("Security checkpoint required")
        return checkpoint(req, cookies, user_agent, username, csrftoken)
    else:
        try:
            regx_error = re.search(r'"message":"(.*?)",', req.text).group(1)
            print(regx_error)
        except:
            print(req.text)
        ask = input("Something went wrong. Try again? [Y/N]: ")
        if ask.lower() == "y":
            return Login()
        else:
            exit()
# تشغيل البرنامج
Login()
