# banking-system

#include <iostream>
#include <vector>
#include <fstream>
#include <string>
#include <thread>
#include <chrono>
#include <iomanip>
#include <stdexcept>
#include <cctype>
#include <algorithm>

using namespace std;

class BankAccount {
protected:
    string accountHolder;
    double balance;
    string pin;
    int accountNumber;
    string email;

public:
    BankAccount(string holder, double initialBalance, string accountPin, int accNum, string emailAddr) 
        : accountHolder(holder), balance(initialBalance), pin(accountPin), accountNumber(accNum), email(emailAddr) {}

    virtual void deposit(double amount) {
        if (amount <= 0) throw invalid_argument("Deposit amount must be positive.");
        balance += amount;
        saveToFile(); // Save after deposit
    }

    virtual void withdraw(double amount) {
        if (amount <= 0) throw invalid_argument("Withdraw amount must be positive.");
        if (amount > balance) throw runtime_error("Insufficient funds.");
        balance -= amount;
        saveToFile(); // Save after withdrawal
    }

    double getBalance() const {
        return balance;
    }

    string getPin() const {
        return pin;
    }

    int getAccountNumber() const {
        return accountNumber;
    }

    string getAccountHolder() const {
        return accountHolder;
    }

    string getEmail() const {
        return email;
    }

    void updateAccountHolder(string newHolder) {
        accountHolder = newHolder;
        saveToFile(); // Save after updating account holder
    }

    void updateEmail(string newEmail) {
        email = newEmail;
        saveToFile(); // Save after updating email
    }

    void updatePin(string newPin) {
        pin = newPin;
        saveToFile(); // Save after updating PIN
    }

    static bool isPinValid(const string& pin) {
        return pin.length() == 4 && all_of(pin.begin(), pin.end(), ::isdigit);
    }

    virtual void displayAccountInfo() const = 0;
    virtual string getAccountType() const = 0;
    virtual void saveToFile() const = 0;
};

class SavingsAccount : public BankAccount {
public:
    SavingsAccount(string holder, string accountPin, int accNum, string emailAddr) 
        : BankAccount(holder, 0.0, accountPin, accNum, emailAddr) {}

    void displayAccountInfo() const override {
        cout << "Account Number: " << accountNumber << endl
             << "Savings Account Holder: " << accountHolder << endl
             << "Balance: ₹" << fixed << setprecision(2) << balance << endl
             << "Email: " << email << endl;
    }

    string getAccountType() const override {
        return "Savings";
    }

    void saveToFile() const override {
        ofstream file("savings_accounts.txt", ios::app);
        file << "Account Holder: " << accountHolder << ", Balance: ₹" << fixed << setprecision(2) << balance 
             << ", Account Number: " << accountNumber << ", Email: " << email << endl;
        file.close();
    }
};

class CurrentAccount : public BankAccount {
public:
    CurrentAccount(string holder, string accountPin, int accNum, string emailAddr) 
        : BankAccount(holder, 0.0, accountPin, accNum, emailAddr) {}

    void displayAccountInfo() const override {
        cout << "Account Number: " << accountNumber << endl;
        cout << "Current Account Holder: " << accountHolder << endl;
        cout << "Balance: ₹" << fixed << setprecision(2) << balance << endl;
        cout << "Email: " << email << endl;
    }

    string getAccountType() const override {
        return "Current";
    }

    void saveToFile() const override {
        ofstream file("current_accounts.txt", ios::app);
        file << "Account Holder: " << accountHolder << ", Balance: ₹" << fixed << setprecision(2) << balance 
             << ", Account Number: " << accountNumber << ", Email: " << email << endl;
        file.close();
    }
};

class Bank {
private:
    vector<BankAccount*> accounts;
    int nextAccountNumber = 10000000; // Start from an 8-digit number

    void saveToAdminFile(const string& holder, const string& pin, int accountNum, const string& email, const string& accountType) {
        ofstream file("admin.txt", ios::app);
        file << "Account Holder: " << holder << ", PIN: " << pin 
             << ", Account Number: " << accountNum << ", Email: " << email 
             << ", Account Type: " << accountType << endl;
        file.close();
    }

    void removeFromAdminFile(int accountNum) {
        vector<string> lines;
        ifstream file("admin.txt");
        string line;

        while (getline(file, line)) {
            if (line.find("Account Number: " + to_string(accountNum)) == string::npos) {
                lines.push_back(line);
            }
        }
        file.close();

        ofstream outFile("admin.txt");
        for (const auto& l : lines) {
            outFile << l << endl;
        }
        outFile.close();
    }

public:
    ~Bank() {
        for (auto account : accounts) {
            account->saveToFile(); // Save before deleting
            delete account;
        }
    }

    void createAccount(string type, string holder, string accountPin, string emailAddr) {
        if (!BankAccount::isPinValid(accountPin)) {
            cout << "PIN must be exactly 4 digits and numeric." << endl;
            return;
        }
        BankAccount* account = nullptr;
        if (type == "savings") {
            account = new SavingsAccount(holder, accountPin, nextAccountNumber++, emailAddr);
        } else if (type == "current") {
            account = new CurrentAccount(holder, accountPin, nextAccountNumber++, emailAddr);
        } else {
            throw invalid_argument("Invalid account type.");
        }
        accounts.push_back(account);

        // Save to admin file with account type
        saveToAdminFile(holder, accountPin, account->getAccountNumber(), emailAddr, account->getAccountType());

        cout << endl << "Creating account..." << endl;
        std::this_thread::sleep_for(std::chrono::seconds(3));
        
        cout << endl << "Account successfully created!" << endl;
        cout << "Account Number: " << account->getAccountNumber() << endl;
        cout << "Press Enter to continue...";
        cin.ignore(numeric_limits<streamsize>::max(), '\n');
        cin.get();
    }

    void listAccounts() const {
        if (accounts.empty()) {
            cout << endl << "No accounts registered." << endl << endl;
            return;
        }
        for (const auto& account : accounts) {
            cout << endl;
            account->displayAccountInfo();
            cout << endl;
        }
    }

    BankAccount* authenticate(int accountNum, string accountPin) {
        cout << endl << "Logging in..." << endl;
        std::this_thread::sleep_for(std::chrono::seconds(3));
        
        for (auto account : accounts) {
            if (account->getAccountNumber() == accountNum && 
                account->getPin() == accountPin) {
                cout << "Account login successful!" << endl;
                return account;
            }
        }
        throw runtime_error("Authentication failed.");
    }

    BankAccount* findAccountByNumber(int accountNum) {
        for (auto account : accounts) {
            if (account->getAccountNumber() == accountNum) {
                return account;
            }
        }
        throw runtime_error("Account not found.");
    }

    void deleteAccount(int accountNum, string accountPin) {
        for (auto it = accounts.begin(); it != accounts.end(); ++it) {
            if ((*it)->getAccountNumber() == accountNum && (*it)->getPin() == accountPin) {
                removeFromAdminFile(accountNum); // Remove from admin file
                delete *it;
                accounts.erase(it);
                cout << "Account successfully deleted!" << endl;
                return;
            }
        }
        throw runtime_error("Account not found or incorrect PIN.");
    }
};

void clearConsole() {
    for (int i = 0; i < 100; ++i) {
        cout << endl;
    }
}

void displayWelcomeMessage() {
    clearConsole();
    const string message = "WELCOME TO BANK MANAGEMENT SYSTEM";
    int terminalWidth = 80;
    int padding = (terminalWidth - message.length()) / 2;

    cout << string(padding, ' ') << message << endl;
    cout << string(padding, ' ') << string(message.length(), '=') << endl << endl;
}

void displayMenu() {
    cout << "\n1. Create Savings Account\n";
    cout << "2. Create Current Account\n";
    cout << "3. List Accounts\n";
    cout << "4. Login to Account\n";
    cout << "5. Find Account by Number\n";
    cout << "6. Delete Account\n";
    cout << "7. Exit\n";
    cout << endl << "Choose an operation: ";
}

void pauseForUser() {
    cout << "Press Enter to continue...";
    cin.ignore(numeric_limits<streamsize>::max(), '\n');
    cin.get();
}

void accountMenu(BankAccount* account) {
    double amount;
    int choice;
    string newHolder, newEmail, newPin;

    do {
        clearConsole();
        cout << "Account Menu" << endl;
        account->displayAccountInfo();
        cout << "1. Deposit\n";
        cout << "2. Withdraw\n";
        cout << "3. Update Account Holder\n";
        cout << "4. Update Email\n";
        cout << "5. Update PIN\n";
        cout << "6. Logout\n";
        cout << "Choose an option: ";
        cin >> choice;

        switch (choice) {
            case 1:
                cout << "Enter amount to deposit: ";
                cin >> amount;
                account->deposit(amount);
                cout << "Deposit successful!" << endl;
                pauseForUser();
                break;
            case 2:
                cout << "Enter amount to withdraw: ";
                cin >> amount;
                try {
                    account->withdraw(amount);
                    cout << "Withdrawal successful!" << endl;
                } catch (const runtime_error& e) {
                    cout << e.what() << endl;
                }
                pauseForUser();
                break;
            case 3:
                cout << "Enter new account holder name: ";
                cin >> newHolder;
                account->updateAccountHolder(newHolder);
                cout << "Account holder name updated!" << endl;
                pauseForUser();
                break;
            case 4:
                cout << "Enter new email: ";
                cin >> newEmail;
                account->updateEmail(newEmail);
                cout << "Email updated!" << endl;
                pauseForUser();
                break;
            case 5:
                cout << "Enter new PIN: ";
                cin >> newPin;
                if (BankAccount::isPinValid(newPin)) {
                    account->updatePin(newPin);
                    cout << "PIN updated!" << endl;
                } else {
                    cout << "Invalid PIN format. Must be 4 digits." << endl;
                }
                pauseForUser();
                break;
            case 6:
                cout << "Logging out..." << endl;
                std::this_thread::sleep_for(std::chrono::seconds(2));
                break;
            default:
                cout << "Invalid option. Please try again." << endl;
                pauseForUser();
        }
    } while (choice != 6);
}

int main() {
    Bank bank;
    int choice;

    do {
        displayWelcomeMessage();
        displayMenu();
        cin >> choice;

        switch (choice) {
            case 1: {
                string holder, pin, email;
                cout << "Enter account holder name: ";
                cin >> holder;
                cout << "Enter 4-digit PIN: ";
                cin >> pin;
                cout << "Enter email: ";
                cin >> email;
                bank.createAccount("savings", holder, pin, email);
                break;
            }
            case 2: {
                string holder, pin, email;
                cout << "Enter account holder name: ";
                cin >> holder;
                cout << "Enter 4-digit PIN: ";
                cin >> pin;
                cout << "Enter email: ";
                cin >> email;
                bank.createAccount("current", holder, pin, email);
                break;
            }
            case 3:
                bank.listAccounts();
                pauseForUser();
                break;
            case 4: {
                int accountNum;
                string accountPin;
                cout << "Enter account number: ";
                cin >> accountNum;
                cout << "Enter PIN: ";
                cin >> accountPin;

                try {
                    BankAccount* account = bank.authenticate(accountNum, accountPin);
                    accountMenu(account);
                } catch (const runtime_error& e) {
                    cout << e.what() << endl;
                    pauseForUser();
                }
                break;
            }
            case 5: {
                int accountNum;
                cout << "Enter account number to find: ";
                cin >> accountNum;
                try {
                    BankAccount* account = bank.findAccountByNumber(accountNum);
                    account->displayAccountInfo();
                } catch (const runtime_error& e) {
                    cout << e.what() << endl;
                }
                pauseForUser();
                break;
            }
            case 6: {
                int accountNum;
                string accountPin;
                cout << "Enter account number to delete: ";
                cin >> accountNum;
                cout << "Enter PIN: ";
                cin >> accountPin;

                try {
                    bank.deleteAccount(accountNum, accountPin);
                } catch (const runtime_error& e) {
                    cout << e.what() << endl;
                }
                pauseForUser();
                break;
            }
            case 7:
                cout << "Exiting program..." << endl;
                std::this_thread::sleep_for(std::chrono::seconds(2));
                break;
            default:
                cout << "Invalid option. Please try again." << endl;
                pauseForUser();
        }
    } while (choice != 7);

    return 0;
}
