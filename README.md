# Abdanix_financial_management_system
"""
financial_system.py
--------------------
Week 3 Task: Modular Financial Management System

"""

import json
import os

DATA_FILE = "transactions.json"


# ---------------------------------------------------------------------
# FILE HANDLING FUNCTIONS
# ---------------------------------------------------------------------

def load_data():
    """
    Loads transactions from the JSON data file.
    Returns a list of transaction dictionaries.
    If the file does not exist or is corrupted, returns an empty list.
    """
    if not os.path.exists(DATA_FILE):
        return []

    try:
        with open(DATA_FILE, "r") as file:
            data = json.load(file)
            if isinstance(data, list):
                return data
            return []
    except (json.JSONDecodeError, IOError):
        print("Warning: Data file could not be read. Starting with an empty list.")
        return []


def save_data(transactions):
    """
    Saves the given list of transactions to the JSON data file.
    Called after every add/update/delete so data is never lost.
    """
    try:
        with open(DATA_FILE, "w") as file:
            json.dump(transactions, file, indent=4)
    except IOError:
        print("Error: Could not save data to file.")


# ---------------------------------------------------------------------
# INPUT VALIDATION HELPERS
# ---------------------------------------------------------------------

def get_valid_number(prompt):
    """
    Keeps asking the user for input until a valid positive number
    (int or float) is entered. Returns the number as a float.
    """
    while True:
        value = input(prompt).strip()
        if value == "":
            print("Input cannot be empty. Please try again.")
            continue
        try:
            number = float(value)
            if number <= 0:
                print("Amount must be greater than zero.")
                continue
            return number
        except ValueError:
            print("Invalid number. Please enter digits only (e.g., 1500 or 1500.50).")


def get_valid_choice(prompt, valid_choices):
    """
    Keeps asking the user for a menu choice until it matches
    one of the valid_choices (a list/tuple of strings).
    """
    while True:
        choice = input(prompt).strip()
        if choice in valid_choices:
            return choice
        print(f"Invalid choice. Please select one of: {', '.join(valid_choices)}")


def get_non_empty_text(prompt):
    """
    Keeps asking until the user provides a non-empty text value.
    """
    while True:
        text = input(prompt).strip()
        if text == "":
            print("This field cannot be empty. Please try again.")
            continue
        return text


def generate_next_id(transactions):
    """
    Generates the next unique transaction ID based on existing data.
    """
    if not transactions:
        return 1
    return max(t["id"] for t in transactions) + 1


# ---------------------------------------------------------------------
# CORE FEATURE FUNCTIONS
# ---------------------------------------------------------------------

def add_transaction(transactions):
    """
    Adds a new transaction (income or expense) to the list.
    Handles its own input collection and validation.
    Returns the updated transactions list.
    """
    print("\n--- Add New Transaction ---")

    t_type = get_valid_choice(
        "Enter type (income/expense): ", ["income", "expense"]
    )
    category = get_non_empty_text("Enter category (e.g., Salary, Food, Rent): ")
    amount = get_valid_number("Enter amount: ")
    description = get_non_empty_text("Enter description: ")

    transaction = {
        "id": generate_next_id(transactions),
        "type": t_type,
        "category": category,
        "amount": amount,
        "description": description,
    }

    transactions.append(transaction)
    save_data(transactions)
    print(f"Transaction added successfully with ID: {transaction['id']}")
    return transactions


def view_transactions(transactions):
    """
    Displays all transactions in a readable, tabular format.
    Also shows a summary of total income, total expense, and balance.
    """
    print("\n--- All Transactions ---")

    if not transactions:
        print("No transactions found.")
        return

    print(f"{'ID':<5}{'Type':<10}{'Category':<15}{'Amount':<12}{'Description'}")
    print("-" * 60)

    total_income = 0
    total_expense = 0

    for t in transactions:
        print(f"{t['id']:<5}{t['type']:<10}{t['category']:<15}{t['amount']:<12.2f}{t['description']}")
        if t["type"] == "income":
            total_income += t["amount"]
        else:
            total_expense += t["amount"]

    print("-" * 60)
    print(f"Total Income : {total_income:.2f}")
    print(f"Total Expense: {total_expense:.2f}")
    print(f"Balance      : {(total_income - total_expense):.2f}")


def find_transaction_by_id(transactions, transaction_id):
    """
    Helper function used by search/update/delete.
    Returns the transaction dict if found, otherwise None.
    """
    for t in transactions:
        if t["id"] == transaction_id:
            return t
    return None


def search_transaction(transactions):
    """
    Searches transactions either by ID or by a keyword found
    in the category or description fields.
    """
    print("\n--- Search Transaction ---")
    mode = get_valid_choice(
        "Search by (id/keyword): ", ["id", "keyword"]
    )

    if mode == "id":
        try:
            search_id = int(input("Enter transaction ID: ").strip())
        except ValueError:
            print("Invalid ID. ID must be a number.")
            return

        result = find_transaction_by_id(transactions, search_id)
        if result:
            print("Transaction found:")
            print(result)
        else:
            print("No transaction found with that ID.")
    else:
        keyword = get_non_empty_text("Enter keyword: ").lower()
        results = [
            t for t in transactions
            if keyword in t["category"].lower() or keyword in t["description"].lower()
        ]
        if results:
            print(f"Found {len(results)} matching transaction(s):")
            for t in results:
                print(t)
        else:
            print("No matching transactions found.")


def update_transaction(transactions):
    """
    Updates an existing transaction's fields based on its ID.
    Returns the updated transactions list.
    """
    print("\n--- Update Transaction ---")

    if not transactions:
        print("No transactions available to update.")
        return transactions

    try:
        update_id = int(input("Enter the ID of the transaction to update: ").strip())
    except ValueError:
        print("Invalid ID. ID must be a number.")
        return transactions

    transaction = find_transaction_by_id(transactions, update_id)

    if transaction is None:
        print("No transaction found with that ID.")
        return transactions

    print("Leave a field blank to keep its current value.")

    new_category = input(f"New category [{transaction['category']}]: ").strip()
    if new_category:
        transaction["category"] = new_category

    new_amount = input(f"New amount [{transaction['amount']}]: ").strip()
    if new_amount:
        try:
            new_amount_value = float(new_amount)
            if new_amount_value > 0:
                transaction["amount"] = new_amount_value
            else:
                print("Amount must be positive. Keeping old value.")
        except ValueError:
            print("Invalid amount entered. Keeping old value.")

    new_description = input(f"New description [{transaction['description']}]: ").strip()
    if new_description:
        transaction["description"] = new_description

    save_data(transactions)
    print("Transaction updated successfully.")
    return transactions


def delete_transaction(transactions):
    """
    Deletes a transaction from the list based on its ID.
    Returns the updated transactions list.
    """
    print("\n--- Delete Transaction ---")

    if not transactions:
        print("No transactions available to delete.")
        return transactions

    try:
        delete_id = int(input("Enter the ID of the transaction to delete: ").strip())
    except ValueError:
        print("Invalid ID. ID must be a number.")
        return transactions

    transaction = find_transaction_by_id(transactions, delete_id)

    if transaction is None:
        print("No transaction found with that ID.")
        return transactions

    confirm = get_valid_choice(
        f"Are you sure you want to delete transaction {delete_id}? (yes/no): ",
        ["yes", "no"]
    )

    if confirm == "yes":
        transactions.remove(transaction)
        save_data(transactions)
        print("Transaction deleted successfully.")
    else:
        print("Deletion cancelled.")

    return transactions


# ---------------------------------------------------------------------
# MENU / PROGRAM FLOW
# ---------------------------------------------------------------------

def display_menu():
    """Displays the main menu options to the user."""
    print("\n===== Financial Management System =====")
    print("1. Add Transaction")
    print("2. View Transactions")
    print("3. Search Transaction")
    print("4. Update Transaction")
    print("5. Delete Transaction")
    print("6. Exit")
    print("=========================================")


def main():
    """
    Main function that controls the overall program flow.
    Loads existing data on start, runs the menu loop, and
    exits cleanly when the user chooses to.
    """
    transactions = load_data()
    print("Welcome to the Financial Management System!")

    while True:
        display_menu()
        choice = get_valid_choice(
            "Select an option (1-6): ", ["1", "2", "3", "4", "5", "6"]
        )

        if choice == "1":
            transactions = add_transaction(transactions)
        elif choice == "2":
            view_transactions(transactions)
        elif choice == "3":
            search_transaction(transactions)
        elif choice == "4":
            transactions = update_transaction(transactions)
        elif choice == "5":
            transactions = delete_transaction(transactions)
        elif choice == "6":
            save_data(transactions)
            print("Data saved. Goodbye!")
            break


if __name__ == "__main__":
    main()
