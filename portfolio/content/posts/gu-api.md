+++ 
date = 2024-01-14T00:59:27+05:30
title = "Revolutionizing University Accessibility: The i-cloud Bot"
description = "Exploring the journey of creating a bot to enhance accessibility to Galgotias University's i-cloud portal."
slug = "revolutionizing-university-accessibility"
authors = ["Ritik Ranjan"]
tags = ["Galgotias University", "Bot", "Web Scraping", "Accessibility"]
categories = ["Technology", "Education"]
externalLink = ""
series = []
+++

## Unveiling "i-cloud": Revolutionizing Accessibility to University Information

In the heart of Galgotias University lies an essential hub for students, faculty, and staff—the i-cloud portal (<https://gu.icloudems.com/>). This Angular app, hosted on AWS, serves as a gateway to crucial information such as timetables, marks, announcements, and more.

### Connectivity Challenges in the Remote Haven

Galgotias University, nestled near the Yamuna Express Highway, grapples with internet connectivity challenges due to its remote location. The result? Slow internet during college hours, making accessing the i-cloud portal a tedious task.

## Enter the WhatsApp Bot: A Solution Born from Necessity

To tackle this issue, my friend and I embarked on a mission to create a WhatsApp bot, designed to respond to specific commands such as /myattendance, /upcoming, /announcement, and /marks. Why WhatsApp? Good thing about whatsapp is. WhatsApp can work on 2G networks, but it may be slow and unreliable. WhatsApp recommends using a 3G or 4G network for the best experience. And whatsapp can still runnable on Android OS 5.0 (I believe everyone now owns a iphone 13 pro max, kinda cheap).

## Building Bridges: Creating an API for i-cloud

Surprisingly, there were no existing APIs to communicate with the i-cloud portal. Undeterred, we pivoted to a web scraping solution. While the legality of scraping is debatable, it became a necessary step in our quest for accessibility.
The Tech Behind the Curtain

Being proficient in Python, we chose Selenium for its ease of use. We also opted for SQLite for a lightweight database, storing student passwords, phone numbers, and admission details.

Building such an API service was extremely easy in Python! We wanted some dedicated framework, so at first we were about to learn Scrapy, but due to a lack of time, we dropped the idea, as we didn't have that much time. So, we chose Selenium (Selenium is an open source umbrella project for a range of tools and libraries aimed at supporting browser automation). I can't say in what language Selenium is written (maybe in Java); what I do know is how to use Selenium with Python.

## Extracting Timetable Data: Navigating the Angular Maze

Timetable extraction required navigating through three main `<table>` elements on a cluttered Angular app. Despite the complexity, Selenium and BeautifulSoup worked in tandem to provide a structured JSON output.

Here are our tables and their schema for storing student passwords, along with a section.

```sql
CREATE TABLE IF NOT EXISTS students_credentials (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    admission_number TEXT NOT NULL,
    password TEXT NOT NULL,
    section INTEGER NOT NULL,

    UNIQUE(admission_number)
);
```

We also need tables for storing phone numbers and admission numbers.

```sql
CREATE A TABLE IF IT DOES NOT EXISTS students_phonenumbers(
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    admission_number TEXT NOT NULL,
    phonenumber INT,
    
    UNIQUE(admission_number)
)
```

Now, the actual deal begins. First,  we need to login with our credentials. This is the function we wrote for the login.

```python
# This is minimal code; for full code, check my github

def login(self) -> FireFoxWebDriver:
    self.driver.get(GU_ICLOUD_EMS_LOGIN)

    self._input_and_click(
        self.driver.find_element(By.ID, config["input_username_id"]),
        self.__admission_number,)

    self._input_and_click(
        self.driver.find_element(By.ID, config["input_password_id"]),
        self.__password,)

    self.driver.execute_script(f"""verify_branch({self.__admission_number!r})""")
    submit = self.driver.find_element(By.ID, config["login_button_id"])
    submit.click()
    self.__login_success = True

    return self.driver
```

Logging in was simple, as was getting timetable data. We just need to click on the timetable button (we were on a phone or laptop).

![image](https://gist.github.com/assets/37435729/fa5f4f43-355d-4a76-add6-810cca411168)

```python
class TimeTableDriver(WebDriver):
    def click_button(self) -> None:
        self._click_timetable_icon()a

    def _click_timetable_icon(self):
        href = "schedulerand/tt_report_view.php"
        wait = WebDriverWait(self.driver, 10)
        wait.until(EC.element_to_be_clickable((By.XPATH, f"//a[@href='{href}']")))
        element = self.driver.find_element (By.XPATH, f"//a[@href='{href}']")

        element.click()
```

We downloaded the raw page source and scraped with BeautifulSoup. Our timetable page was very clumsy (I hate you, Angular App).
The timetable page consists of three main `<table/>` The first table is the actual main table; here's a picture.

![image](https://gist.github.com/assets/37435729/2d4fa925-f133-45a3-81aa-1d122efff234)

And here are the other two:

![image](https://gist.github.com/assets/37435729/a0267d91-f775-48a7-a3e0-150f7e6b4afc)

This is the main code for getting timetable data.

```python
def get_timetable(self) -> Arrangement:
    table = self.get_timetable_details()
    data: Arrangement = {
        "Mon": [],
        "Tue": [],
        "Wed": [],
        "Thu": [],
        "Fri": [],
        "Sat": [],
        "Sun": [],
    }
    date_range = self.get_date_range()
    for tr in table.find_all("tr")[1:]:
        day: str = tr.find("td").text.strip()  # type: ignore

        if day not in data:
            data[day] = []

        date = date_range[day]
        start_time, end_time = tr.find_all("td")[1].text.strip().split("-")

        start_time_object = date + timedelta(
            hours=int(start_time.split(":")[0]),
            minutes=int(start_time.split(":")[1]),
        )

        end_time_object = date + timedelta(
            hours=int(end_time.split(":")[0]),
            minutes=int(end_time.split(":")[1]),
        )

        data[day].append(
            {
                "date": TimeTableParser._datetime_to_sqlite_string(date),
                "start_time": TimeTableParser._datetime_to_sqlite_string(
                    start_time_object
                ),
                "end_time": TimeTableParser._datetime_to_sqlite_string(
                    end_time_object
                ),
                "faculty_name": tr.find_all("td")[2].text.strip(),
                "slot": _SlotParser(tr.find_all("td")[3].text.strip()).to_dict(),
                "class": self.class_name,
            }
        )

    return data
```

At the end, this is what we get:

```json
{
    "timetable": {
        "Mon": [
            {
                "date": "...",
                "start_time": "...",
                "end_time": "...",
                "faculty_name": "...",
                "slot": {
                    "course_name": "...",
                    "course_type": "...",
                    "course_code": "...",
                    "section": ...,
                    "room": "...",
                    "block": "..."
                },
                "class": "..."
            },
            {
                ...
            },
            ...
        ],
        ...
    },
    "alternative_timetable": {
        "Sun": [
            {
                "date": "...",
                "start_time": "...",
                "end_time": "...",
                "faculty_name": "...",
                "alternate_faculty_name": "...",
                "slot": {
                    "course_name": "...",
                    "course_type": "...",
                    "course_code": "...",
                    "section": ...,
                    "room": "",
                    "block": ""
                },
                "class": "..."
            },
            ...
        ]
    }
}
```

This JSON is much more readable, easier to understand, and easier to deal with.

## Challenges Faced and Lessons Learned

While challenges persisted, such as incomplete attendance data retrieval, the journey taught us the importance of adaptable solutions and the power of leveraging existing technologies like WhatsApp.

## Conclusion: A Glimpse into the Future

Our endeavor continues as we refine the bot, striving for a seamless user experience. This project reflects not only a response to connectivity challenges but also a testament to our problem-solving skills and the innovative use of technology.

## The Flowchart of Connectivity

![Diagram](https://gist.github.com/assets/37435729/53650bf9-67d2-4abc-adce-7b951e13fbbf)

In this intricate dance between HTML parsers and bots, communication via API becomes the linchpin of accessibility, transforming the i-cloud portal into a realm of possibilities.

... and that's it. We have successfully scraped the timetable data from the i-cloud portal. Now, we can use this data to create a WhatsApp bot, a Telegram bot, or even a Discord bot. We can also use this data to create a web app, a mobile app, or even a desktop app. The possibilities are endless.

PEACE
