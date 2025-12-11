import csv
import json
import os
from datetime import datetime
 
PATIENT_FILE = "patients.csv"
BACKUP_FILE = "backup.json"
AUDIT_LOG = "audit.log"
 
# LOAD PATIENT RECORDS FROM CSV

def load_patients():
    patients = {}
    if not os.path.exists(PATIENT_FILE):
        print("patients.csv not found. Starting with empty records.")
        return patients
    try:
        with open(PATIENT_FILE, "r") as file:
            reader = csv.DictReader(file)
            for row in reader:
                pid = int(row["id"])
                patients[pid] = {
                    "name": row["name"],
                    "diagnosis": row["diagnosis"],
                    "medications": row["medications"].split("|")
                }
    except Exception as e:
        print("Error while loading CSV:", e)
    return patients
 
 
# AUDIT LOGGING
def log_action(message):
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    with open(AUDIT_LOG, "a") as log:
        log.write(f"[{timestamp}] {message}\n")
 
# APPOINTMENT SYSTEM
# appointments = [(date, time, doctor, patient_id)]
appointments = []
 
def schedule_appointment(date, time, doctor, patient_id, patients):
    if patient_id not in patients:
        print("Error: Patient ID not found.")
        log_action(f"FAILED scheduling for invalid patient {patient_id}")
        return
 
    # Check overlapping appointments
    for appt in appointments:
        if appt[0] == date and appt[1] == time and appt[2] == doctor:
            print("Error: Doctor already has an appointment at this time.")
            log_action(f"FAILED overlap appointment for patient {patient_id}")
            return
 
    new_appt = (date, time, doctor, patient_id)
    appointments.append(new_appt)
    print("Appointment scheduled!")
    log_action(f"Appointment scheduled: {new_appt}")
 
 
def cancel_appointment(patient_id):
    removed = False
    for appt in appointments:
        if appt[3] == patient_id:
            appointments.remove(appt)
            print("Appointment canceled.")
            log_action(f"Appointment canceled for ID {patient_id}")
            removed = True
            break
    if not removed:
        print("No appointment found for this patient.")
        log_action(f"FAILED cancellation attempt for ID {patient_id}")

#Appointments

def appointment_conflict(date: str, time: str, doctor: str) -> bool:
    for ap in appointments:
        ap_date, ap_time, ap_doc, ap_pid =ap
        if ap_date == date and ap_time == time and ap_doc.lower() == doctor.lower():
            return True
        return False
 
 
 
# TREATMENT REPORT (GROUP BY DIAGNOSIS)
def generate_treatment_report(patients):
    report = {}
 
    for pid, info in patients.items():
        diag = info["diagnosis"]
        if diag not in report:
            report[diag] = []
        report[diag].append((pid, info["name"], info["medications"]))
 
    print("\nTreatment Report:")
    for diag, data in report.items():
        print(f"\nDiagnosis: {diag}")
        for entry in data:
            print(f"ID: {entry[0]}, Name: {entry[1]}, Medications: {entry[2]}")
 
    log_action("Generated treatment report")
 
    return report
 
# BACKUP TO JSON WITH ERROR HANDLING
def backup_data(patients):
    try:
        with open(BACKUP_FILE, "w") as file:
            json.dump({"patients": patients, "appointments": appointments}, file, indent=4)
        print("Backup created successfully.")
        log_action("Backup created.")
    except Exception as e:
        print("Backup failed:", e)
        log_action("FAILED backup")
 
def load_backup():
    if not os.path.exists(BACKUP_FILE):
        print("No backup file found.")
        return
    try:
        with open(BACKUP_FILE, "r") as file:
            data = json.load(file)
            print("Backup loaded.")
            return data
    except json.JSONDecodeError:
        print("Backup corrupt. Unable to load.")
        log_action("FAILED backup load (corrupt)")
 
# ROLLBACK LAST 3 ACTIONS
def rollback_last_actions(patients):
    if not os.path.exists(AUDIT_LOG):
        print("No audit log available.")
        return
    with open(AUDIT_LOG, "r") as file:
        lines = file.readlines()
    last_actions = lines[-3:]  # last 3 actions
    print("\n ROLLBACK STARTED")
 
    for action in reversed(last_actions):  # reverse to undo in correct order
        if "Appointment scheduled:" in action:
            # undo by removing last appointment
            if appointments:
                removed = appointments.pop()
                print(f"Undo appointment → {removed}")
        elif "Appointment canceled for ID" in action:
            # can't restore details → simply log rollback
            print("Undo cancel appointment (no restore data available).")
        elif "Backup created" in action:
            print("Undo backup creation (no file deletion for safety).")
 
    log_action("Rollback of last 3 actions executed")
 
# MAIN
def main():
    patients = load_patients()
 
    while True:
        print("\n Hospital Patient Management")
        print("1. Schedule Appointment")
        print("2. Cancel Appointment")
        print("3. Show Treatment Report")
        print("4. show appoitments")
        print("5. Backup Data")
        print("6. Rollback Last 3 Actions")
        print("7. Exit")
 
        choice = input("Enter choice: ")
 
        if choice == "1":
            date = input("Enter date (YYYY-MM-DD): ")
            time = input("Enter time (HH:MM): ")
            doctor = input("Enter doctor name: ")
            pid = int(input("Enter patient ID: "))
            schedule_appointment(date, time, doctor, pid, patients)
 
        elif choice == "2":
            pid = int(input("Enter patient ID to cancel appointment: "))
            cancel_appointment(pid)
 
        elif choice == "3":
            generate_treatment_report(patients)
        elif choice == "4":
                if not appointments:
                    print("[INFO] No appointments scheduled.")
                else:
                    print("Appointments:")
                    for ap in sorted(appointments, key=lambda x: (x[0], x[1], x[2])):
                        print(f"  Date: {ap[0]} Time: {ap[1]} Doctor: {ap[2]} Patient ID: {ap[3]}")
 
        elif choice == "5":
            backup_data(patients)
 
        elif choice == "6":
            rollback_last_actions(patients)
 
        elif choice == "7":
            print("Exiting system...")
            break
        else:
            print("Invalid choice.")
 
 
if __name__ == "__main__":
    main()
