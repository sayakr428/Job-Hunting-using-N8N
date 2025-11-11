<img width="1361" height="253" alt="image" src="https://github.com/user-attachments/assets/d0cc0770-4f46-4c0b-b1c1-1332d700317f" />


# 🧠 AI-Powered Job Application Assistant  
Automate your entire job search workflow using **n8n**, **OpenAI API**, **Google Sheets**, and **Telegram**.  
This project intelligently scans LinkedIn for matching roles, analyzes job descriptions against your resume, generates custom cover letters, and alerts you instantly when a great match appears.  

---

## 🚀 Features  

- 🔍 **Automated LinkedIn Search** – Runs every day at 5 PM and fetches relevant job postings  
- 🧾 **Resume Intelligence** – Extracts data from your resume (PDF on Google Drive)  
- 🤖 **AI Matching & Cover Letters** – Uses OpenAI GPT to score jobs and write tailored cover letters  
- 🗂️ **Google Sheets Integration** – Stores all job data (title, company, score, cover letter, etc.)  
- 💬 **Telegram Notifications** – Sends alerts for high-scoring opportunities directly to your phone  

---

## 🧩 Tech Stack  

| Component | Purpose |
|------------|----------|
| **n8n** | Workflow orchestration & automation |
| **Google Drive API** | Resume storage & retrieval |
| **Google Sheets API** | Store search filters and job results |
| **OpenAI API** | Resume-job matching & cover letter generation |
| **Telegram Bot API** | Push notifications for shortlisted jobs |
| **LinkedIn HTML Parsing** | Extract job details such as title, company, description, and job ID |

---

## ⚙️ Workflow Overview  

### Step 1:  
Runs automatically every day at **5 PM** via the **Schedule Trigger** node.  

### Step 2: Resume Extraction  
Downloads your resume from **Google Drive** → extracts the text for analysis.  

### Step 3: Job Filters  
Reads your search criteria from a Google Sheet (columns like `Keyword`, `Location`, `Experience Level`, `Remote`, `Job Type`, `Easy Apply`).  

### Step 4: LinkedIn Search URL Builder  
A **Code node** converts your filters into a dynamic LinkedIn search URL:  

```js
let url = "https://www.linkedin.com/jobs/search/?f_TPR=r86400";

const keyword = $input.first().json.Keyword;
const location = $input.first().json.Location;
const experienceLevel = $input.first().json['Experience Level'];
const remote = $input.first().json.Remote;
const jobType = $input.first().json['Job Type'];
const easyApply = $input.first().json['Easy Apply'];

if (keyword) url += `&keywords=${keyword}`;
if (location) url += `&location=${location}`;

// Experience Level Mapping
if (experienceLevel) {
  const transformedExperiences = experienceLevel
    .split(",")
    .map((exp) => {
      switch (exp.trim()) {
        case "Internship": return "1";
        case "Entry level": return "2";
        case "Associate": return "3";
        case "Mid-Senior level": return "4";
        case "Director": return "5";
        case "Executive": return "6";
        default: return "";
      }
    })
    .filter(Boolean);
  url += `&f_E=${transformedExperiences.join(",")}`;
}

// Remote / Hybrid / Onsite Mapping
if (remote) {
  const transformedRemote = remote
    .split(",")
    .map((e) => {
      switch (e.trim()) {
        case "Remote": return "2";
        case "Hybrid": return "3";
        case "On-Site": return "1";
        default: return "";
      }
    })
    .filter(Boolean);
  url += `&f_WT=${transformedRemote.join(",")}`;
}

// Job Type Mapping
if (jobType) {
  const transformedJobType = jobType
    .split(",")
    .map((type) => type.trim().charAt(0).toUpperCase());
  url += `&f_JT=${transformedJobType.join(",")}`;
}

// Easy Apply Filter
if (easyApply) url += "&f_EA=true";

return { url };
```
## 🧮 Step 5: Job Collection

### Fetch Jobs from LinkedIn**
- **Add Node:** HTTP Request  
- **Method:** `GET`  
- **URL:** `={{ $json.url }}`  
- This retrieves the raw LinkedIn job listings HTML.  

---

### **Step 6: Extract Job Links**
- **Add Node:** HTML Extract  
- **Operation:** Extract HTML Content  
- **Extraction Values:**

| Key | CSS Selector | Return Value | Attribute |
|------|---------------|--------------|------------|
| links | `ul.jobs-search__results-list li div a[class*="base-card"]` | Attribute | `href` |

✅ Enable **Return Array**  

---

### **Step 7: Split Out Job Links**
- **Add Node:** Split Out  
- **Field to Split Out:** `links`  
- This converts the array of links into individual job items.  

---

### **Step 8: Loop Over Items**
- **Add Node:** Loop Over Items  
- **Batch Size:** `1`  
- Processes each job one by one to avoid overload.  

---

### **Step 9: Wait Node**
- **Add Node:** Wait  
- **Mode:** After Time Interval  
- **Wait:** `8 seconds`  
- Prevents rate limiting between LinkedIn requests.  

---

### **Step 10: Fetch Job Page**
- **Add Node:** HTTP Request  
- **URL:** `={{ $json.links }}`  
- Fetches the full LinkedIn job description HTML for each listing.  

---

### **Step 11: Parse Job Attributes**
- **Add Node:** HTML Extract  
- **Configure the following fields:**

| Key | CSS Selector | Return Value | Attribute |
|------|---------------|--------------|------------|
| title | `div h1` | Text | - |
| company | `div span a` | Text | - |
| location | `div span[class*='topcard__flavor topcard__flavor--bullet']` | Text | - |
| description | `div.description__text.description__text--rich` | Text | - |
| jobid | `a[data-item-type='semaphore']` | Attribute | `data-semaphore-content-urn` |

---

### **Step 12: Modify Job Attributes**
- **Add Node:** Edit Fields (Set)  
- Clean and format job data:  

| Name | Type | Value |
|------|------|--------|
| description | String | `={{ $json.description.replaceAll(/\s+/g, " ") }}` |
| jobid | String | `={{ $json.jobid.split(":").pop() }}` |
| applyLink | String | `={{ "https://www.linkedin.com/jobs/view/" + $json.jobid.split(":").pop() }}` |

✅ Enable **Include Other Fields**  

---

## 🧠 AI-Powered Resume Matching

### **Step 13: Connect OpenAI API**
- Add a **Google PaLM / OpenAI API** credential in n8n.  
- Recommended models: `gpt-4-turbo` or `gpt-3.5-turbo`.  

---

### **Step 14: Add AI Agent Node**
- **Prompt Type:** Define Below  
- **Prompt:**
  ```
  You are an assistant that helps match candidates to job opportunities. Your task is to review my resume, then analyze both the resume and job description to calculate a job matching score. Additionally, create a cover letter based on my resume and the job description. The cover letter should contain at least 2 paragraphs and should exclude the name, address, and signature sections from the beginning and end.
  When using special characters like quotation marks ("), ensure they are properly escaped with a backslash. The output must be formatted as valid JSON that can be parsed without errors. For example your response should be like: {"score": 80, "coverLetter": "sample cover letter" }
  job_description: {{ $json.description }}
  my_resume: {{ $('Edit Fields').item.json.resumeText }} ```

### **Step 15: Parse AI Output**  
- **Add Node:** Edit Fields (Set)  
- **Purpose:** Cleans and extracts valid JSON from the AI response.  
- **Field:** JSON Output  
```js
{{ $json.output.replaceAll(/```(?:json)?/g, "") }}
```

✅ Enable **Include Other Fields**

---

## 📊 Save to Google Sheets  

### **Step 16: Append or Update Row**  
- **Add Node:** Google Sheets  
- **Operation:** Append or Update  
- **Document:** Select your spreadsheet  
- **Sheet:** `Result`  

**Mapping:**

| Column | Value |
|---------|--------|
| Title | `={{ $('Modify Job Attributes').item.json.title }}` |
| Company | `={{ $('Modify Job Attributes').item.json.company }}` |
| Location | `={{ $('Modify Job Attributes').item.json.location }}` |
| Link | `={{ $('Modify Job Attributes').item.json.applyLink }}` |
| Description | `={{ $('Modify Job Attributes').item.json.description }}` |
| Score | `={{ $json.score }}` |
| Cover Letter | `={{ $json.coverLetter }}` |

✅ **Matching Column:** `link`  
Automatically appends new jobs or updates existing ones if the link already exists.

---

## 📬 Telegram Notifications  

### **Step 17: Score Filter**  
- **Add Node:** IF  
- **Condition:**

| Field | Operator | Value |
|--------|-----------|--------|
| `={{ $json.score }}` | Larger or Equal | `50` |

This filters and sends only high-scoring job opportunities to Telegram.

---

### **Step 18: Send Telegram Message**  
- **Add Node:** Telegram (Resource: Message)  
- **Chat ID:** Your Telegram user ID  
- **Text:**
```text
🔥 New Matching Job Found!

Title: {{ $('Modify Job Attributes').item.json.title }}
Company: {{ $('Modify Job Attributes').item.json.company }}
Location: {{ $('Modify Job Attributes').item.json.location }}
Score: {{ $json.score }}
Apply: {{ $('Modify Job Attributes').item.json.applyLink }}

Cover letter available in Google Sheet 📄
```

✅ Disable **“Append n8n Attribution”**  
This sends personalized job alerts directly to your Telegram account.  

---

## 🔁 Step 19: Close the Loop  

- **Goal:** Ensure the workflow processes all job listings continuously.  
- **Action:** Connect both branches from the **Score Filter** node:  
  - **True →** Loop Over Items  
  - **False →** Loop Over Items  

This setup allows the workflow to keep iterating through every job posting, regardless of the score threshold, ensuring that all listings are analyzed, scored, and stored.  

✅ Once connected, your entire automation loop is complete — from fetching jobs → scoring them with AI → saving results → sending Telegram alerts → and looping back for the next listing.  
