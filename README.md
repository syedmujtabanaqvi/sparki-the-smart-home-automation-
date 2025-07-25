# sparki-the-smart-home-automation-
Sparky is a Python voice assistant that controls Arduino (lights, temperature), chats with AI, sends WhatsApp messages, and logs chats to MongoDB — all with voice commands. A smart combo of AI and IoT.
import pyttsx3
import speech_recognition as sr
import datetime
import pywhatkit
import wikipedia
import webbrowser
import pyautogui
import pymongo
from groq import Groq
import serial
import time

# ==================== Arduino Setup ====================
try:
    arduino = serial.Serial(port='COM6', baudrate=9600, timeout=2)
    time.sleep(2)
    arduino_connected = True
except:
    arduino_connected = False

def send_command_to_arduino(cmd):
    if arduino_connected:
        arduino.flushInput()
        arduino.write((cmd + '\n').encode())
        print(f"Sent to Arduino: {cmd}")

        if cmd == "temp":
            time.sleep(2)
            try:
                if arduino.in_waiting:
                    response = arduino.readline().decode().strip()
                    print("Received:", response)
                    temp_value = float(response)
                    speak(f"The temperature is {temp_value} degrees Celsius.")
                else:
                    speak("No response from Arduino.")
            except ValueError:
                speak("Invalid temperature data received from Arduino.")
        else:
            time.sleep(0.5)
            if arduino.in_waiting:
                response = arduino.readline().decode().strip()
                print("Arduino:", response)
            speak(f"Lights turned {cmd} successfully.")
    else:
        speak("Arduino not connected.")

def map_command(query):
    if any(phrase in query for phrase in ["turn on light", "on light", "lights on", "turn on", "on"]):
        return "on"
    if any(phrase in query for phrase in ["turn off light", "lights off", "turn off", "off"]):
        return "off"
    if "temperature" in query:
        return "temp"
    if any(phrase in query for phrase in ["exit", "stop"]):
        return "exit"
    return ""

# ==================== Voice Engine ====================
engine = pyttsx3.init("sapi5")
voices = engine.getProperty("voices")
engine.setProperty("voice", voices[0].id)
engine.setProperty("rate", 170)

def speak(audio):
    print("Sparky:", audio)
    engine.say(audio)
    engine.runAndWait()

# ==================== Take Voice Command ====================
def takeCommand():
    r = sr.Recognizer()
    with sr.Microphone() as source:
        print("Listening...")
        r.pause_threshold = 1
        r.energy_threshold = 900
        r.adjust_for_ambient_noise(source)
        try:
            audio = r.listen(source, timeout=5)
        except sr.WaitTimeoutError:
            print("Listening timed out.")
            return "none"
    try:
        print("Recognizing...")
        query = r.recognize_google(audio, language='en-in')
        print(f"You: {query}")
        return query.lower()
    except sr.UnknownValueError:
        speak("Sorry, I didn't catch that.")
        return "none"
    except sr.RequestError:
        speak("Speech service is unavailable.")
        return "none"

# ==================== MongoDB Setup ====================
mongo_client = pymongo.MongoClient("mongodb://localhost:27017/")
db = mongo_client['assistant_db']
chatlog_collection = db['chatlogs']

def save_to_db(role, content):
    chatlog_collection.insert_one({
        "role": role,
        "content": content,
        "timestamp": datetime.datetime.utcnow()
    })

def get_last_messages(limit=1000):
    cursor = chatlog_collection.find().sort("timestamp", -1).limit(limit)
    messages = list(cursor)
    messages.reverse()
    return messages

# ==================== Chatbot Setup ====================
Username = "mujtaba"
Assistantname = "sparky"
GROQAPIKey = "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"
client = Groq(api_key=GROQAPIKey)

def RealTimeInformation():
    now = datetime.datetime.now()
    return (
        f"Day: {now.strftime('%A')}\n"
        f"Date: {now.strftime('%d')}\n"
        f"Month: {now.strftime('%B')}\n"
        f"Year: {now.strftime('%Y')}\n"
        f"Time: {now.strftime('%H:%M:%S')}\n"
    )

def AnswerModifier(answer):
    return '\n'.join([line for line in answer.split('\n') if line.strip()])

def ChatBot(query):
    if any(x in query for x in ["turn off chat bot", "turn off chat", "exit"]):
        speak("Okay, exiting chatbot mode.")
        return "exit_chatbot"
    try:
        save_to_db("user", query)
        rows = get_last_messages(1000)
        messages = [{"role": "system", "content": RealTimeInformation()}]
        for row in rows:
            messages.append({
                "role": row.get("role", "user"),
                "content": row.get("content", "")
            })
        messages.append({"role": "user", "content": query})

        completion = client.chat.completions.create(
            model="llama3-70b-8192",
            messages=messages,
            max_tokens=1024,
            temperature=0.7,
            top_p=1,
            stream=True
        )

        answer = ""
        for chunk in completion:
            if chunk.choices[0].delta.content:
                answer += chunk.choices[0].delta.content

        answer = AnswerModifier(answer.replace("<\\s>", ""))
        save_to_db("assistant", answer)
        speak(answer)
        return answer

    except Exception as e:
        speak("Something went wrong with the chatbot.")
        print("Chatbot Error:", e)
        return f"Error: {e}"

# ==================== Basic Functions ====================
def greetMe():
    hour = int(datetime.datetime.now().hour)
    if hour < 12:
        speak("Good morning, sir")
    elif hour < 18:
        speak("Good afternoon, sir")
    else:
        speak("Good evening, sir")
    speak("Please tell me, how can I help you?")

def searchGoogle(query):
    speak("This is what I found on Google.")
    query = query.replace("google", "")
    pywhatkit.search(query)
    try:
        result = wikipedia.summary(query, 1)
        speak(result)
    except:
        speak("No speakable output available.")

def searchYoutube(query):
    speak("This is what I found for your search!")
    query = query.replace("youtube", "")
    webbrowser.open("https://www.youtube.com/results?search_query=" + query)
    pywhatkit.playonyt(query)
    speak("Done, Sir.")

def searchWikipedia(query):
    speak("Searching from Wikipedia....")
    query = query.replace("wikipedia", "")
    results = wikipedia.summary(query, sentences=2)
    speak("According to Wikipedia")
    speak(results)

def send_whatsapp_message(query):
    contacts = {
        "father": "+92",
        "ali": "+92",
    }
    def take_message():
        r = sr.Recognizer()
        with sr.Microphone() as source:
            speak("Listening for message...")
            try:
                audio = r.listen(source, timeout=5)
                return r.recognize_google(audio)
            except:
                speak("I didn't catch the message.")
                return ""
    for name in contacts:
        if name in query:
            speak(f"What message to send to {name}?")
            message = take_message()
            if message:
                pywhatkit.sendwhatmsg_instantly(contacts[name], message, wait_time=10, tab_close=True)
                speak(f"Sent message to {name}")
            return
    speak("This name is not in the contact list.")

# ==================== Main Program ====================
if __name__ == "__main__":
    active = True
    greetMe()
    while active:
        query = takeCommand()
        if query == "none":
            continue

        cmd = map_command(query)
        if cmd == "exit":
            speak("Goodbye, sir!")
            active = False
        elif cmd:
            send_command_to_arduino(cmd)
        elif "hello" in query:
            speak("Hello sir, how are you?")
        elif "i am fine" in query:
            speak("That's great, sir.")
        elif "how are you" in query:
            speak("Perfect, sir.")
        elif "thank you" in query:
            speak("You're welcome, sir.")
        elif "google" in query:
            searchGoogle(query)
        elif "youtube" in query:
            searchYoutube(query)
        elif "wikipedia" in query:
            searchWikipedia(query)
        elif "the time" in query:
            strTime = datetime.datetime.now().strftime("%H:%M")
            speak(f"Sir, the time is {strTime}")
        elif "volume up" in query:
            pyautogui.press("volumeup")
        elif "volume down" in query:
            pyautogui.press("volumedown")
        elif "volume mute" in query:
            pyautogui.press("volumemute")
        elif "send message" in query:
            send_whatsapp_message(query)
        elif "chatbot" in query:
            speak("Chatbot mode activated. Say 'turn off chat bot' to exit.")
            while True:
                user_input = takeCommand()
                if user_input == "none":
                    continue
                result = ChatBot(user_input)
                if result == "exit_chatbot":
                    break
        else:
            speak("Please say a valid command.")

    if arduino_connected:
        arduino.close()
        print("Arduino connection closed.")
