from flask import Flask, request, jsonify
import os
import json
import subprocess
import openai
import sqlite3
from datetime import datetime
import pytesseract
from PIL import Image

app = Flask(__name__)

# Configure OpenAI API Key
openai.api_key = "your-openai-key"

# Task Mapping
TASK_FUNCTIONS = {
    "A1": "install_and_run_datagen",
    "A2": "format_markdown",
    "A3": "count_wednesdays",
    "A4": "sort_contacts",
    "A5": "extract_recent_logs",
    "A6": "generate_markdown_index",
    "A7": "extract_email_sender",
    "A8": "extract_credit_card_number",
    "A9": "find_similar_comments",
    "A10": "calculate_gold_ticket_sales"
}

@app.route('/run', methods=['POST'])
def run_task():
    task_desc = request.args.get('task')
    if not task_desc:
        return jsonify({"error": "No task provided"}), 400

    try:
        # Step 1: Use LLM to determine task ID
        task_id = parse_task_with_llm(task_desc)
        if task_id not in TASK_FUNCTIONS:
            return jsonify({"error": "Unknown task"}), 400

        # Step 2: Execute the corresponding function
        task_func = globals().get(TASK_FUNCTIONS[task_id])
        if not task_func:
            return jsonify({"error": "Task function not implemented"}), 500

        result = task_func()
        return jsonify({"result": result}), 200

    except Exception as e:
        return jsonify({"error": str(e)}), 500

@app.route('/read', methods=['GET'])
def read_file():
    file_path = request.args.get("path")
    if not file_path or not os.path.exists(file_path):
        return "", 404
    try:
        with open(file_path, "r") as file:
            return file.read(), 200
    except Exception:
        return "", 500

def parse_task_with_llm(task_desc):
    """Uses LLM to determine the task ID."""
    prompt = f"Given the following task description, determine which task ID it belongs to: {task_desc}\n"
    response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=[{"role": "system", "content": "You are an AI that maps task descriptions to task IDs (A1 to A10)."},
                  {"role": "user", "content": prompt}]
    )
    return response["choices"][0]["message"]["content"].strip()
def install_and_run_datagen():
    user_email = os.getenv("USER_EMAIL", "test@example.com")
    
    # Install uv if not installed
    subprocess.run(["pip", "install", "uv"], check=True)

    # Download and run script
    script_url = "https://raw.githubusercontent.com/sanand0/tools-in-data-science-public/tds-2025-01/project-1/datagen.py"
    subprocess.run(["curl", "-o", "datagen.py", script_url], check=True)
    subprocess.run(["python", "datagen.py", user_email], check=True)

    return "datagen.py executed successfully"
def format_markdown():
    subprocess.run(["npx", "prettier@3.4.2", "--write", "/data/format.md"], check=True)
    return "Formatted markdown file successfully"
def count_wednesdays():
    count = 0
    with open("/data/dates.txt", "r") as file:
        for line in file:
            date_obj = datetime.strptime(line.strip(), "%Y-%m-%d")
            if date_obj.weekday() == 2:  # Wednesday
                count += 1
    with open("/data/dates-wednesdays.txt", "w") as out_file:
        out_file.write(str(count))
    return f"Counted {count} Wednesdays"
def sort_contacts():
    with open("/data/contacts.json", "r") as file:
        contacts = json.load(file)
    sorted_contacts = sorted(contacts, key=lambda x: (x["last_name"], x["first_name"]))
    with open("/data/contacts-sorted.json", "w") as out_file:
        json.dump(sorted_contacts, out_file, indent=4)
    return "Contacts sorted successfully"
def extract_credit_card_number():
    img = Image.open("/data/credit-card.png")
    text = pytesseract.image_to_string(img)
    card_number = "".join(filter(str.isdigit, text))
    with open("/data/credit-card.txt", "w") as file:
        file.write(card_number)
    return "Extracted credit card number"


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
