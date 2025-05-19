import streamlit as st
import pandas as pd
from fpdf import FPDF
from twilio.rest import Client
import smtplib
from email.mime.text import MIMEText
from datetime import datetime

# -----------------------------------------------
# PERSONAL INTRO & CONTEXT (STATIC HEADER)
# -----------------------------------------------
st.set_page_config(page_title="Mduh's Billing System", layout="wide")
st.title("Mduh's Billing & Collection Tracker")
st.markdown("""
> This app was created by Mduh after 3 unrelated debtors **independently** approached him for financial help.  
> This is not a loan-shark operation. They came to me, not the other way around.  
> This app serves to track behavior, enforce fair agreements, and protect the lender from manipulation or ghosting.

---  
""")

# -----------------------------------------------
# USER INFO SECTION
# -----------------------------------------------
st.sidebar.header("Your Info")
your_name = st.sidebar.text_input("Your Name", value="Mduh")
your_id = st.sidebar.text_input("ID Number", value="ID12345678")
your_phone = st.sidebar.text_input("Your Contact", value="+27...")
your_bank = st.sidebar.text_input("Your Bank", value="ABSA")
your_account = st.sidebar.text_input("Account Number", value="00012345678")

# -----------------------------------------------
# DEBTOR SETUP
# -----------------------------------------------
st.header("Debtor Overview")

debtors = {
    "Debtor A": {
        "start_date": "2024-01-18",
        "bank": "Capitec", "account": "1234567890", "phone": "+27...", "email": "debtorA@example.com",
        "loan": 10000, "due_day": 5, "payments": []
    },
    "Debtor B": {
        "start_date": "2024-01-18",
        "bank": "FNB", "account": "9876543210", "phone": "+27...", "email": "debtorB@example.com",
        "loan": 8000, "due_day": 10, "payments": []
    },
    "Debtor C": {
        "start_date": "2024-01-31",
        "bank": "Standard Bank", "account": "4567890123", "phone": "+27...", "email": "debtorC@example.com",
        "loan": 6000, "due_day": 15, "payments": []
    }
}

# Store in session
if "debtor_data" not in st.session_state:
    st.session_state.debtor_data = debtors

# -----------------------------------------------
# HELPER FUNCTIONS
# -----------------------------------------------
def calculate_behavior(payments):
    if not payments:
        return "No Data"
    late = sum(1 for p in payments if p["status"] == "Late")
    missed = sum(1 for p in payments if p["status"] == "Missed")
    total = len(payments)
    if missed / total > 0.5:
        return "Highly Unresponsive"
    elif late / total > 0.5:
        return "Unresponsive"
    elif late / total > 0.3:
        return "Neutral"
    elif late == 0:
        return "Highly Responsive"
    else:
        return "Responsive"

def send_email(subject, body, to_email):
    try:
        msg = MIMEText(body)
        msg["Subject"] = subject
        msg["From"] = "your_email@example.com"
        msg["To"] = to_email
        with smtplib.SMTP_SSL("smtp.gmail.com", 465) as server:
            server.login("your_email@example.com", "your_password")
            server.send_message(msg)
        return True
    except:
        return False

def send_sms(body, to_number):
    try:
        client = Client("TWILIO_SID", "TWILIO_TOKEN")
        client.messages.create(
            body=body,
            from_="+1234567890",
            to=to_number
        )
        return True
    except:
        return False

def generate_pdf(debtor_name, data, remarks):
    pdf = FPDF()
    pdf.add_page()
    pdf.set_font("Arial", size=12)
    pdf.cell(200, 10, txt=f"Clause & Effect Agreement - {debtor_name}", ln=True)
    pdf.multi_cell(0, 10, txt=f"Initial Loan: R{data['loan']}\nStart Date: {data['start_date']}")
    pdf.multi_cell(0, 10, txt=f"Debtor Bank: {data['bank']}, Account: {data['account']}")
    pdf.multi_cell(0, 10, txt="Remarks: " + remarks)
    for p in data["payments"]:
        pdf.cell(200, 10, txt=f"{p['date']} - R{p['amount']} ({p['status']})", ln=True)
    filename = f"{debtor_name.replace(' ', '_')}_agreement.pdf"
    pdf.output(filename)
    return filename
    # -----------------------------------------------
# MAIN LOOP - Debtor Management
# -----------------------------------------------
for name, data in st.session_state.debtor_data.items():
    st.subheader(f"{name} - Total Borrowed: R{data['loan']}")
    st.write(f"Start Date: {data['start_date']} | Due Day Each Month: {data['due_day']}")

    col1, col2 = st.columns(2)
    with col1:
        amount = st.number_input(f"Log Payment for {name}", key=f"{name}_amt", min_value=0.0)
    with col2:
        pay_date = st.date_input(f"Payment Date", key=f"{name}_date", value=datetime.today())

    if st.button(f"Add Payment for {name}"):
        due_date = datetime(pay_date.year, pay_date.month, data["due_day"])
        status = "On Time" if pay_date <= due_date else ("Late" if (pay_date - due_date).days <= 10 else "Missed")
        data["payments"].append({"amount": amount, "date": pay_date.strftime("%Y-%m-%d"), "status": status})
        st.success("Payment added.")

    if data["payments"]:
        df = pd.DataFrame(data["payments"])
        st.write(df)
        st.markdown(f"**Behavior Rating:** {calculate_behavior(data['payments'])}")
        st.markdown("---")

        if st.button(f"Generate Clause & Effect PDF for {name}"):
            remarks = st.text_area(f"Remarks for {name}", key=f"{name}_remarks")
            filename = generate_pdf(name, data, remarks)
            with open(filename, "rb") as f:
                st.download_button("Download Agreement PDF", f, file_name=filename)

        if st.button(f"Send SMS Reminder to {name}"):
            success = send_sms(f"Hi {name}, please remember to make your payment.", data["phone"])
            st.success("SMS sent!" if success else "Failed to send SMS.")

        if st.button(f"Send Email Reminder to {name}"):
            success = send_email("Payment Reminder", f"Hi {name}, this is a reminder to make your monthly payment.", data["email"])
            st.success("Email sent!" if success else "Failed to send email.")

    else:
        st.info("No payments logged yet.")

# -----------------------------------------------
# Visual Summary (End of Page)
# -----------------------------------------------
st.markdown("## Summary Dashboard")
summary = []
for name, data in st.session_state.debtor_data.items():
    total_paid = sum(p["amount"] for p in data["payments"])
    balance = data["loan"] - total_paid
    summary.append({"Debtor": name, "Loan": data["loan"], "Paid": total_paid, "Outstanding": balance})
df_summary = pd.DataFrame(summary)
st.dataframe(df_summary)
