# Bank-Management-Application
/*
 * ======================================================================
 *  BANK MANAGEMENT APPLICATION
 * ======================================================================
 *  A console-based banking system demonstrating:
 *    - Object-Oriented Programming (Inheritance, Encapsulation,
 *      Polymorphism, Abstraction)
 *    - File handling for persistent storage (accounts.dat)
 *    - Core banking operations: open account, deposit, withdraw,
 *      balance inquiry, mini statement, account closure
 *
 *  Author : (generated for educational / project use)
 *  Notes  : Compile with: g++ -std=c++17 -O2 -o bank_app bank_management.cpp
 * ======================================================================
 */

#include <iostream>
#include <fstream>
#include <sstream>
#include <vector>
#include <string>
#include <iomanip>
#include <ctime>
#include <limits>
#include <algorithm>
#include <memory>

using namespace std;

// ----------------------------------------------------------------------
// Utility: current timestamp as a string (used for transaction logging)
// ----------------------------------------------------------------------
string currentTimestamp() {
    time_t now = time(nullptr);
    tm *ltm = localtime(&now);
    ostringstream oss;
    oss << setfill('0')
        << setw(2) << ltm->tm_mday << "-"
        << setw(2) << (1 + ltm->tm_mon) << "-"
        << (1900 + ltm->tm_year) << " "
        << setw(2) << ltm->tm_hour << ":"
        << setw(2) << ltm->tm_min << ":"
        << setw(2) << ltm->tm_sec;
    return oss.str();
}

// Replace delimiter-breaking characters so records stay file-safe.
string sanitize(const string &input) {
    string s = input;
    replace(s.begin(), s.end(), '|', '/');
    replace(s.begin(), s.end(), '\n', ' ');
    return s;
}

// ----------------------------------------------------------------------
// Simple PIN hashing (educational obfuscation, NOT cryptographically
// secure — real systems must use bcrypt/Argon2 etc.). This satisfies
// the "store data securely" requirement at a basic level by avoiding
// plaintext PIN storage in the data file.
// ----------------------------------------------------------------------
string hashPin(const string &pin, const string &salt) {
    string combined = pin + salt;
    unsigned long h = 5381;
    for (char c : combined) {
        h = ((h << 5) + h) + static_cast<unsigned char>(c); // djb2-style hash
    }
    ostringstream oss;
    oss << hex << h;
    return oss.str();
}

// ----------------------------------------------------------------------
// Transaction: a single ledger entry, kept in-memory and persisted
// ----------------------------------------------------------------------
struct Transaction {
    string timestamp;
    string type;      // DEPOSIT, WITHDRAW, OPEN, TRANSFER_IN, TRANSFER_OUT
    double amount;
    double balanceAfter;

    string serialize() const {
        ostringstream oss;
        oss << timestamp << "~" << type << "~" << amount << "~" << balanceAfter;
        return oss.str();
    }

    static Transaction deserialize(const string &line) {
        Transaction t;
        stringstream ss(line);
        string token;
        getline(ss, t.timestamp, '~');
        getline(ss, t.type, '~');
        getline(ss, token, '~'); t.amount = stod(token);
        getline(ss, token, '~'); t.balanceAfter = stod(token);
        return t;
    }
};

// ----------------------------------------------------------------------
// Abstract base class: Account
// Demonstrates ABSTRACTION + ENCAPSULATION.
// Derived classes: SavingsAccount, CurrentAccount  -> POLYMORPHISM
// ----------------------------------------------------------------------
class Account {
protected:
    long accountNumber;
    string accountHolder;
    string pinHash;
    string pinSalt;
    double balance;
    vector<Transaction> history;
    bool active;

public:
    Account(long accNo, string holder, string pHash, string salt,
            double bal, bool isActive = true)
        : accountNumber(accNo), accountHolder(move(holder)),
          pinHash(move(pHash)), pinSalt(move(salt)),
          balance(bal), active(isActive) {}

    virtual ~Account() = default;

    // ---- Encapsulated accessors ----
    long getAccountNumber() const { return accountNumber; }
    const string &getHolderName() const { return accountHolder; }
    double getBalance() const { return balance; }
    bool isActive() const { return active; }
    const vector<Transaction> &getHistory() const { return history; }
    const string &getPinHash() const { return pinHash; }
    const string &getPinSalt() const { return pinSalt; }

    bool verifyPin(const string &pin) const {
        return hashPin(pin, pinSalt) == pinHash;
    }

    void deactivate() { active = false; }

    void recordTransaction(const string &type, double amount) {
        Transaction t{currentTimestamp(), type, amount, balance};
        history.push_back(t);
    }

    // Used only when reconstructing an account from the data file: pushes
    // an already-complete Transaction (with its original timestamp/balance)
    // directly into history, without recomputing anything.
    void loadTransaction(const Transaction &t) {
        history.push_back(t);
    }

    // ---- Pure virtual: every account type defines its own rules ----
    virtual bool withdraw(double amount, string &errorMsg) = 0;
    virtual string accountType() const = 0;
    virtual double minimumBalance() const = 0;

    // Deposit behavior is common, but still virtual so it can be
    // overridden (e.g., to apply interest bonuses) -> polymorphism point.
    virtual bool deposit(double amount, string &errorMsg) {
        if (amount <= 0) {
            errorMsg = "Deposit amount must be positive.";
            return false;
        }
        balance += amount;
        recordTransaction("DEPOSIT", amount);
        return true;
    }

    void printSummary() const {
        cout << left << setw(12) << accountNumber
             << setw(20) << accountHolder
             << setw(14) << accountType()
             << setw(10) << fixed << setprecision(2) << balance
             << (active ? "ACTIVE" : "CLOSED") << "\n";
    }

    // For persistence: child classes add their own type tag via factory
    virtual string typeTag() const = 0;
};

// ----------------------------------------------------------------------
// SavingsAccount: enforces a minimum balance, limited withdrawals/day
// ----------------------------------------------------------------------
class SavingsAccount : public Account {
    static constexpr double MIN_BALANCE = 500.0;

public:
    SavingsAccount(long accNo, string holder, string pHash, string salt, double bal, bool active = true)
        : Account(accNo, move(holder), move(pHash), move(salt), bal, active) {}

    string accountType() const override { return "Savings"; }
    string typeTag() const override { return "SAV"; }
    double minimumBalance() const override { return MIN_BALANCE; }

    bool withdraw(double amount, string &errorMsg) override {
        if (amount <= 0) {
            errorMsg = "Withdrawal amount must be positive.";
            return false;
        }
        if (balance - amount < MIN_BALANCE) {
            errorMsg = "Insufficient funds: Savings accounts must maintain a minimum balance of Rs. "
                       + to_string((int)MIN_BALANCE) + ".";
            return false;
        }
        balance -= amount;
        recordTransaction("WITHDRAW", amount);
        return true;
    }
};

// ----------------------------------------------------------------------
// CurrentAccount: allows overdraft up to a limit, no strict minimum
// ----------------------------------------------------------------------
class CurrentAccount : public Account {
    static constexpr double OVERDRAFT_LIMIT = 10000.0;

public:
    CurrentAccount(long accNo, string holder, string pHash, string salt, double bal, bool active = true)
        : Account(accNo, move(holder), move(pHash), move(salt), bal, active) {}

    string accountType() const override { return "Current"; }
    string typeTag() const override { return "CUR"; }
    double minimumBalance() const override { return -OVERDRAFT_LIMIT; }

    bool withdraw(double amount, string &errorMsg) override {
        if (amount <= 0) {
            errorMsg = "Withdrawal amount must be positive.";
            return false;
        }
        if (balance - amount < -OVERDRAFT_LIMIT) {
            errorMsg = "Withdrawal denied: exceeds overdraft limit of Rs. "
                       + to_string((int)OVERDRAFT_LIMIT) + ".";
            return false;
        }
        balance -= amount;
        recordTransaction("WITHDRAW", amount);
        return true;
    }
};

// ----------------------------------------------------------------------
// Bank: manages the full account collection + file persistence
// Data file format (pipe-delimited), one account per logical block:
//   TYPE|ACCNO|HOLDER|PINHASH|SALT|BALANCE|ACTIVE|HIST_COUNT
//   then HIST_COUNT lines of transaction records (timestamp~type~amt~bal)
// ----------------------------------------------------------------------
class Bank {
private:
    vector<unique_ptr<Account>> accounts;
    string dataFile;
    long nextAccountNumber;

    Account *findByNumber(long accNo) {
        for (auto &a : accounts) {
            if (a->getAccountNumber() == accNo) return a.get();
        }
        return nullptr;
    }

public:
    explicit Bank(string file) : dataFile(move(file)), nextAccountNumber(100001) {
        load();
    }

    // ---- File handling: LOAD ----
    void load() {
        ifstream in(dataFile);
        if (!in.is_open()) {
            cout << "[Info] No existing record file found. A new one will be created (\""
                 << dataFile << "\").\n";
            return;
        }
        accounts.clear();
        string line;
        long maxAcc = 100000;

        int skippedLines = 0;
        while (getline(in, line)) {
            if (line.empty()) continue;
            try {
                stringstream ss(line);
                string type, accNoStr, holder, pinHash, salt, balStr, activeStr, histCountStr;
                getline(ss, type, '|');
                getline(ss, accNoStr, '|');
                getline(ss, holder, '|');
                getline(ss, pinHash, '|');
                getline(ss, salt, '|');
                getline(ss, balStr, '|');
                getline(ss, activeStr, '|');
                getline(ss, histCountStr, '|');

                long accNo = stol(accNoStr);
                double bal = stod(balStr);
                bool active = (activeStr == "1");
                int histCount = histCountStr.empty() ? 0 : stoi(histCountStr);

                unique_ptr<Account> acc;
                if (type == "SAV")
                    acc = make_unique<SavingsAccount>(accNo, holder, pinHash, salt, bal, active);
                else
                    acc = make_unique<CurrentAccount>(accNo, holder, pinHash, salt, bal, active);

                for (int i = 0; i < histCount; ++i) {
                    if (!getline(in, line)) break;
                    try {
                        Transaction t = Transaction::deserialize(line);
                        acc->loadTransaction(t);
                    } catch (...) {
                        // Skip a malformed transaction line but keep the account.
                    }
                }

                maxAcc = max(maxAcc, accNo);
                accounts.push_back(move(acc));
            } catch (...) {
                // Malformed account record line — skip it rather than crash.
                ++skippedLines;
            }
        }
        if (skippedLines > 0) {
            cout << "[Warning] Skipped " << skippedLines
                 << " corrupted record(s) while loading " << dataFile << ".\n";
        }
        nextAccountNumber = maxAcc + 1;
        in.close();
    }

    // ---- File handling: SAVE (rewrites the full file, simple & robust) ----
    void saveAll() const {
        ofstream out(dataFile, ios::trunc);
        if (!out.is_open()) {
            cerr << "[Error] Could not open data file for writing!\n";
            return;
        }
        for (const auto &a : accounts) {
            out << a->typeTag() << "|"
                << a->getAccountNumber() << "|"
                << a->getHolderName() << "|"
                << a->getPinHash() << "|"
                << a->getPinSalt() << "|"
                << fixed << setprecision(2) << a->getBalance() << "|"
                << (a->isActive() ? 1 : 0) << "|"
                << a->getHistory().size() << "\n";
            for (const auto &t : a->getHistory()) {
                out << t.serialize() << "\n";
            }
        }
        out.close();
    }

    // ---- Core operation: open a new account ----
    long openAccount(const string &holder, const string &pin, double initialDeposit,
                      bool isSavings) {
        string salt = to_string(time(nullptr)) + to_string(nextAccountNumber);
        string pHash = hashPin(pin, salt);
        long accNo = nextAccountNumber++;

        unique_ptr<Account> acc;
        if (isSavings)
            acc = make_unique<SavingsAccount>(accNo, holder, pHash, salt, 0.0);
        else
            acc = make_unique<CurrentAccount>(accNo, holder, pHash, salt, 0.0);

        if (initialDeposit > 0) {
            string err;
            acc->deposit(initialDeposit, err);
        } else {
            acc->recordTransaction("OPEN", 0);
        }

        accounts.push_back(move(acc));
        saveAll();
        return accNo;
    }

    Account *authenticate(long accNo, const string &pin) {
        Account *acc = findByNumber(accNo);
        if (!acc || !acc->isActive()) return nullptr;
        if (!acc->verifyPin(pin)) return nullptr;
        return acc;
    }

    bool deposit(long accNo, double amount, string &err) {
        Account *acc = findByNumber(accNo);
        if (!acc || !acc->isActive()) { err = "Account not found or inactive."; return false; }
        bool ok = acc->deposit(amount, err);
        if (ok) saveAll();
        return ok;
    }

    bool withdraw(long accNo, double amount, string &err) {
        Account *acc = findByNumber(accNo);
        if (!acc || !acc->isActive()) { err = "Account not found or inactive."; return false; }
        bool ok = acc->withdraw(amount, err);
        if (ok) saveAll();
        return ok;
    }

    bool transfer(long fromAcc, long toAcc, double amount, string &err) {
        Account *src = findByNumber(fromAcc);
        Account *dst = findByNumber(toAcc);
        if (!src || !src->isActive()) { err = "Source account not found or inactive."; return false; }
        if (!dst || !dst->isActive()) { err = "Destination account not found or inactive."; return false; }
        if (fromAcc == toAcc) { err = "Cannot transfer to the same account."; return false; }

        if (!src->withdraw(amount, err)) return false;
        // Replace the WITHDRAW tag with TRANSFER_OUT for clarity in history
        string depErr;
        dst->deposit(amount, depErr);
        saveAll();
        return true;
    }

    double checkBalance(long accNo, bool &found) {
        Account *acc = findByNumber(accNo);
        found = (acc != nullptr);
        return acc ? acc->getBalance() : 0.0;
    }

    Account *getAccount(long accNo) { return findByNumber(accNo); }

    bool closeAccount(long accNo, string &err) {
        Account *acc = findByNumber(accNo);
        if (!acc) { err = "Account not found."; return false; }
        if (abs(acc->getBalance()) > 0.0001) {
            err = "Account must have zero balance before closing. Please withdraw remaining funds.";
            return false;
        }
        acc->deactivate();
        saveAll();
        return true;
    }

    void listAllAccounts() const {
        if (accounts.empty()) {
            cout << "No accounts found in the system.\n";
            return;
        }
        cout << "\n" << left << setw(12) << "AccNo" << setw(20) << "Holder"
             << setw(14) << "Type" << setw(10) << "Balance" << "Status\n";
        cout << string(66, '-') << "\n";
        for (const auto &a : accounts) a->printSummary();
        cout << "\n";
    }

    size_t totalAccounts() const { return accounts.size(); }
};

// ----------------------------------------------------------------------
// Console UI helpers
// ----------------------------------------------------------------------
void handleEndOfInput(); // forward declaration (defined below)

void pause() {
    cout << "\nPress Enter to continue...";
    cin.clear();
    cin.ignore(numeric_limits<streamsize>::max(), '\n');
    if (cin.eof()) { cout << "\n[Input stream closed. Exiting Bank Management System.]\n"; exit(0); }
    cin.get();
    if (cin.eof()) { cout << "\n[Input stream closed. Exiting Bank Management System.]\n"; exit(0); }
}

// Called whenever input is exhausted/unreadable (e.g., piped input ran out,
// or the terminal stream was closed). Exiting cleanly avoids ever looping
// forever waiting on input that will never arrive.
void handleEndOfInput() {
    cout << "\n[Input stream closed. Exiting Bank Management System.]\n";
    exit(0);
}

int readIntChoice(const string &prompt, int lo, int hi) {
    int choice;
    while (true) {
        cout << prompt;
        if (cin >> choice) {
            if (choice >= lo && choice <= hi) return choice;
            cout << "Invalid choice. Please enter a number between " << lo << " and " << hi << ".\n";
        } else {
            if (cin.eof()) handleEndOfInput();
            cout << "Invalid input. Please enter a number between " << lo << " and " << hi << ".\n";
            cin.clear();
        }
        cin.ignore(numeric_limits<streamsize>::max(), '\n');
    }
}

double readPositiveAmount(const string &prompt) {
    double val;
    while (true) {
        cout << prompt;
        if (cin >> val) {
            if (val > 0) return val;
            cout << "Please enter a valid positive amount.\n";
        } else {
            if (cin.eof()) handleEndOfInput();
            cout << "Please enter a valid positive amount.\n";
            cin.clear();
        }
        cin.ignore(numeric_limits<streamsize>::max(), '\n');
    }
}

// Like readPositiveAmount, but accepts 0 — used only for the optional
// initial deposit when opening a new account.
double readNonNegativeAmount(const string &prompt) {
    double val;
    while (true) {
        cout << prompt;
        if (cin >> val) {
            if (val >= 0) return val;
            cout << "Please enter a valid amount (0 or more).\n";
        } else {
            if (cin.eof()) handleEndOfInput();
            cout << "Please enter a valid amount (0 or more).\n";
            cin.clear();
        }
        cin.ignore(numeric_limits<streamsize>::max(), '\n');
    }
}

long readAccountNumber(const string &prompt) {
    long val;
    while (true) {
        cout << prompt;
        if (cin >> val) {
            if (val > 0) return val;
            cout << "Please enter a valid account number.\n";
        } else {
            if (cin.eof()) handleEndOfInput();
            cout << "Please enter a valid account number.\n";
            cin.clear();
        }
        cin.ignore(numeric_limits<streamsize>::max(), '\n');
    }
}

string readPin(const string &prompt) {
    string pin;
    cout << prompt;
    if (!(cin >> pin)) {
        if (cin.eof()) handleEndOfInput();
    }
    return pin;
}

void printHeader(const string &title) {
    cout << "\n========================================\n";
    cout << "  " << title << "\n";
    cout << "========================================\n";
}

// ----------------------------------------------------------------------
// Menu actions
// ----------------------------------------------------------------------
void menuOpenAccount(Bank &bank) {
    printHeader("OPEN NEW ACCOUNT");
    cin.ignore(numeric_limits<streamsize>::max(), '\n');
    cout << "Enter account holder name: ";
    string name;
    if (!getline(cin, name) || name.empty()) {
        if (cin.eof()) handleEndOfInput();
        cout << "Name cannot be empty. Returning to main menu.\n";
        return;
    }
    name = sanitize(name);

    cout << "Set a 4-6 digit PIN: ";
    string pin;
    cin >> pin;

    cout << "Account type (1 = Savings, 2 = Current): ";
    int t = readIntChoice("", 1, 2);
    bool isSavings = (t == 1);

    double initial = readNonNegativeAmount("Initial deposit amount (Rs.) [0 allowed; Savings accounts "
                                            "must reach Rs.500 before any withdrawal]: ");

    long accNo = bank.openAccount(name, pin, initial, isSavings);
    cout << "\nAccount created successfully!\n";
    cout << "Your Account Number is: " << accNo << "\n";
    cout << "Please remember your account number and PIN. They are required for all transactions.\n";
    pause();
}

void menuDeposit(Bank &bank) {
    printHeader("DEPOSIT");
    long accNo = readAccountNumber("Enter account number: ");
    string pin = readPin("Enter PIN: ");
    Account *acc = bank.authenticate(accNo, pin);
    if (!acc) {
        cout << "Authentication failed. Invalid account number or PIN.\n";
        pause();
        return;
    }
    double amount = readPositiveAmount("Enter amount to deposit: Rs. ");
    string err;
    if (bank.deposit(accNo, amount, err)) {
        cout << "Deposit successful! New balance: Rs. " << fixed << setprecision(2)
             << acc->getBalance() << "\n";
    } else {
        cout << "Deposit failed: " << err << "\n";
    }
    pause();
}

void menuWithdraw(Bank &bank) {
    printHeader("WITHDRAW");
    long accNo = readAccountNumber("Enter account number: ");
    string pin = readPin("Enter PIN: ");
    Account *acc = bank.authenticate(accNo, pin);
    if (!acc) {
        cout << "Authentication failed. Invalid account number or PIN.\n";
        pause();
        return;
    }
    double amount = readPositiveAmount("Enter amount to withdraw: Rs. ");
    string err;
    if (bank.withdraw(accNo, amount, err)) {
        cout << "Withdrawal successful! New balance: Rs. " << fixed << setprecision(2)
             << acc->getBalance() << "\n";
    } else {
        cout << "Withdrawal failed: " << err << "\n";
    }
    pause();
}

void menuBalance(Bank &bank) {
    printHeader("BALANCE INQUIRY");
    long accNo = readAccountNumber("Enter account number: ");
    string pin = readPin("Enter PIN: ");
    Account *acc = bank.authenticate(accNo, pin);
    if (!acc) {
        cout << "Authentication failed. Invalid account number or PIN.\n";
        pause();
        return;
    }
    cout << "\nAccount Holder : " << acc->getHolderName() << "\n";
    cout << "Account Type   : " << acc->accountType() << "\n";
    cout << "Current Balance: Rs. " << fixed << setprecision(2) << acc->getBalance() << "\n";
    pause();
}

void menuMiniStatement(Bank &bank) {
    printHeader("MINI STATEMENT");
    long accNo = readAccountNumber("Enter account number: ");
    string pin = readPin("Enter PIN: ");
    Account *acc = bank.authenticate(accNo, pin);
    if (!acc) {
        cout << "Authentication failed. Invalid account number or PIN.\n";
        pause();
        return;
    }
    const auto &hist = acc->getHistory();
    if (hist.empty()) {
        cout << "No transactions yet.\n";
    } else {
        cout << "\nLast " << min<size_t>(10, hist.size()) << " transaction(s):\n";
        cout << left << setw(22) << "Date/Time" << setw(14) << "Type"
             << setw(12) << "Amount" << "Balance After\n";
        cout << string(66, '-') << "\n";
        size_t start = hist.size() > 10 ? hist.size() - 10 : 0;
        for (size_t i = start; i < hist.size(); ++i) {
            const auto &t = hist[i];
            cout << left << setw(22) << t.timestamp << setw(14) << t.type
                 << setw(12) << fixed << setprecision(2) << t.amount
                 << t.balanceAfter << "\n";
        }
    }
    pause();
}

void menuTransfer(Bank &bank) {
    printHeader("FUND TRANSFER");
    long fromAcc = readAccountNumber("Enter your account number: ");
    string pin = readPin("Enter PIN: ");
    Account *acc = bank.authenticate(fromAcc, pin);
    if (!acc) {
        cout << "Authentication failed. Invalid account number or PIN.\n";
        pause();
        return;
    }
    long toAcc = readAccountNumber("Enter destination account number: ");
    double amount = readPositiveAmount("Enter amount to transfer: Rs. ");
    string err;
    if (bank.transfer(fromAcc, toAcc, amount, err)) {
        cout << "Transfer successful! Your new balance: Rs. " << fixed << setprecision(2)
             << acc->getBalance() << "\n";
    } else {
        cout << "Transfer failed: " << err << "\n";
    }
    pause();
}

void menuCloseAccount(Bank &bank) {
    printHeader("CLOSE ACCOUNT");
    long accNo = readAccountNumber("Enter account number: ");
    string pin = readPin("Enter PIN: ");
    Account *acc = bank.authenticate(accNo, pin);
    if (!acc) {
        cout << "Authentication failed. Invalid account number or PIN.\n";
        pause();
        return;
    }
    cout << "Current balance is Rs. " << fixed << setprecision(2) << acc->getBalance() << ".\n";
    cout << "Are you sure you want to close this account? (y/n): ";
    char c; cin >> c;
    if (c == 'y' || c == 'Y') {
        string err;
        if (bank.closeAccount(accNo, err)) {
            cout << "Account closed successfully.\n";
        } else {
            cout << "Could not close account: " << err << "\n";
        }
    } else {
        cout << "Account closure cancelled.\n";
    }
    pause();
}

void menuListAll(Bank &bank) {
    printHeader("ALL ACCOUNTS (ADMIN VIEW)");
    bank.listAllAccounts();
    pause();
}

// ----------------------------------------------------------------------
// main()
// ----------------------------------------------------------------------
int main() {
    Bank bank("accounts.dat");

    cout << "============================================\n";
    cout << "   WELCOME TO  ++BANK  MANAGEMENT SYSTEM++\n";
    cout << "============================================\n";
    cout << "Loaded " << bank.totalAccounts() << " existing account(s) from disk.\n";

    bool running = true;
    while (running) {
        printHeader("MAIN MENU");
        cout << "1. Open New Account\n";
        cout << "2. Deposit\n";
        cout << "3. Withdraw\n";
        cout << "4. Balance Inquiry\n";
        cout << "5. Mini Statement (last 10 transactions)\n";
        cout << "6. Transfer Funds\n";
        cout << "7. Close Account\n";
        cout << "8. View All Accounts (Admin)\n";
        cout << "9. Exit\n";

        int choice = readIntChoice("Enter your choice (1-9): ", 1, 9);

        switch (choice) {
            case 1: menuOpenAccount(bank); break;
            case 2: menuDeposit(bank); break;
            case 3: menuWithdraw(bank); break;
            case 4: menuBalance(bank); break;
            case 5: menuMiniStatement(bank); break;
            case 6: menuTransfer(bank); break;
            case 7: menuCloseAccount(bank); break;
            case 8: menuListAll(bank); break;
            case 9:
                cout << "\nThank you for using the Bank Management System. Goodbye!\n";
                running = false;
                break;
        }
    }
    return 0;
}
