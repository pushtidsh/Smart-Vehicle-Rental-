# Smart-Vehicle-Rental-
A C++ console-based Smart Vehicle Rental System that manages vehicle availability, customer details, rental bookings, returns, and rental charges. It uses Object-Oriented Programming and file handling to simplify rental operations and maintain records efficiently.



#include <iostream>
#include <iomanip>
#include <string>
#include <vector>
#include <fstream>
#include <sstream>
#include <algorithm>
#include <ctime>
#include <cctype>
#include <limits>

using namespace std;

const string ACTIVE_FILE = "active_rentals.txt";
const string HISTORY_FILE = "rental_history.txt";
const string INVOICE_FILE = "invoices.txt";

// ================= GENERAL FUNCTIONS =================

void line(char c = '=', int n = 76) {
    cout << string(n, c) << '\n';
}

void title(const string& text) {
    cout << '\n';
    line();
    cout << "  " << text << '\n';
    line();
}

string readLine(const string& prompt) {
    string value;
    cout << prompt;
    getline(cin, value);
    return value;
}

int readInt(const string& prompt, int low, int high) {
    while (true) {
        cout << prompt;
        string s;
        getline(cin, s);

        stringstream ss(s);
        int value;
        char extra;

        if (ss >> value && !(ss >> extra) &&
            value >= low && value <= high)
            return value;

        cout << "Invalid input. Enter a number from "
             << low << " to " << high << ".\n";
    }
}

double readDouble(const string& prompt, double low, double high) {
    while (true) {
        cout << prompt;
        string s;
        getline(cin, s);

        stringstream ss(s);
        double value;
        char extra;

        if (ss >> value && !(ss >> extra) &&
            value >= low && value <= high)
            return value;

        cout << "Invalid amount. Please try again.\n";
    }
}

// Names: letters and spaces only
string readName(const string& prompt) {
    while (true) {
        string name;
        cout << prompt;
        getline(cin, name);

        bool valid = !name.empty();
        bool hasLetter = false;

        for (char c : name) {
            if (isalpha(static_cast<unsigned char>(c))) {
                hasLetter = true;
            } else if (c != ' ') {
                valid = false;
                break;
            }
        }

        if (valid && hasLetter)
            return name;

        cout << "Invalid name! Use letters and spaces only.\n";
    }
}

// Mobile: exactly 10 digits
string readPhone(const string& prompt) {
    while (true) {
        string phone;
        cout << prompt;
        getline(cin, phone);

        bool valid = phone.length() == 10;

        for (char c : phone) {
            if (!isdigit(static_cast<unsigned char>(c))) {
                valid = false;
                break;
            }
        }

        if (valid)
            return phone;

        cout << "Invalid mobile number! Enter exactly 10 digits.\n";
    }
}

string readLicense(const string& prompt) {
    while (true) {
        string value;
        cout << prompt;
        getline(cin, value);

        if (!value.empty() &&
            value.find('|') == string::npos)
            return value;

        cout << "Invalid licence number. Try again.\n";
    }
}

string currentTime() {
    time_t now = time(nullptr);
    tm* local = std::localtime(&now);

    if (!local)
        return "Time unavailable";

    char buffer[32];
    strftime(buffer, sizeof(buffer),
             "%d-%m-%Y %H:%M:%S", local);

    return buffer;
}

string money(double amount) {
    ostringstream out;
    out << "Rs. " << fixed << setprecision(2) << amount;
    return out.str();
}

vector<string> splitRecord(const string& row) {
    vector<string> fields;
    string part;
    stringstream ss(row);

    while (getline(ss, part, '|'))
        fields.push_back(part);

    return fields;
}

// ================= VEHICLE CLASS =================

class Vehicle {
public:
    int id;
    string name;
    string category;
    string fuel;
    int seats;
    double hourlyRate;
    double dailyRate;
    double deposit;
    bool available;

    Vehicle(int i, string n, string c, string f,
            int s, double hr, double dr, double dep)
        : id(i), name(n), category(c), fuel(f), seats(s),
          hourlyRate(hr), dailyRate(dr), deposit(dep),
          available(true) {}

    void display() const {
        cout << left
             << setw(6) << id
             << setw(31) << name
             << setw(14) << category
             << setw(8) << seats
             << setw(14) << fixed << setprecision(2) << hourlyRate
             << setw(12) << (available ? "Available" : "Rented")
             << '\n';
    }

    double calculateRent(int hours) const {
        int days = hours / 24;
        int remaining = hours % 24;

        double cost = days * dailyRate;

        if (remaining > 0)
            cost += min(remaining * hourlyRate, dailyRate);

        if (hours >= 168)
            cost *= 0.80;
        else if (hours >= 72)
            cost *= 0.90;

        return cost;
    }
};

// ================= RENTAL CLASS =================

class Rental {
public:
    int rentalId = 0;
    int vehicleId = 0;
    string vehicleName;
    string customerName;
    string phone;
    string license;
    int bookedHours = 0;
    double rent = 0;
    double deposit = 0;
    string bookingTime;
};

// ================= RENTAL SYSTEM =================

class SmartRentalSystem {
private:
    vector<Vehicle> vehicles;
    vector<Rental> activeRentals;
    int nextRentalId = 1001;

    // Initial vehicle catalogue
    void loadVehicles() {
        vehicles.emplace_back(101, "Honda Activa 6G",
            "Scooter", "Petrol", 2, 25, 350, 1000);
        vehicles.emplace_back(102, "TVS Jupiter",
            "Scooter", "Petrol", 2, 28, 380, 1000);
        vehicles.emplace_back(103, "TVS Ntorq 125",
            "Scooter", "Petrol", 2, 35, 480, 1500);

        vehicles.emplace_back(201, "Honda SP 125",
            "Bike", "Petrol", 2, 35, 450, 1500);
        vehicles.emplace_back(202, "Royal Enfield Classic 350",
            "Bike", "Petrol", 2, 75, 950, 3000);
        vehicles.emplace_back(203, "Yamaha R15",
            "Bike", "Petrol", 2, 65, 850, 3000);

        vehicles.emplace_back(301, "Maruti Swift",
            "Hatchback", "Petrol", 5, 120, 1800, 5000);
        vehicles.emplace_back(302, "Hyundai i20",
            "Hatchback", "Petrol", 5, 140, 2100, 5000);

        vehicles.emplace_back(401, "Honda City",
            "Sedan", "Petrol", 5, 200, 3000, 8000);
        vehicles.emplace_back(402, "Hyundai Verna",
            "Sedan", "Petrol", 5, 220, 3300, 8000);

        vehicles.emplace_back(501, "Hyundai Creta",
            "SUV", "Petrol", 5, 250, 3800, 10000);
        vehicles.emplace_back(502, "Toyota Innova Crysta",
            "SUV", "Diesel", 7, 350, 5200, 15000);
    }

    Vehicle* findVehicle(int id) {
        for (auto& v : vehicles)
            if (v.id == id)
                return &v;

        return nullptr;
    }

    // Read rental IDs from completed history as well as active rentals.
    // This prevents IDs being reused after a vehicle is returned.
    void loadNextRentalId() {
        int maxId = 1000;
        string row;

        ifstream history(HISTORY_FILE);

        while (getline(history, row)) {
            vector<string> fields = splitRecord(row);

            if (!fields.empty()) {
                try {
                    maxId = max(maxId, stoi(fields[0]));
                } catch (...) {}
            }
        }

        ifstream active(ACTIVE_FILE);

        while (getline(active, row)) {
            vector<string> fields = splitRecord(row);

            if (!fields.empty()) {
                try {
                    maxId = max(maxId, stoi(fields[0]));
                } catch (...) {}
            }
        }

        nextRentalId = maxId + 1;
    }

    // Restore active bookings from the saved file.
    void loadActiveRentals() {
        ifstream in(ACTIVE_FILE);
        string row;

        while (getline(in, row)) {
            vector<string> f = splitRecord(row);

            if (f.size() != 10)
                continue;

            try {
                Rental r;
                r.rentalId = stoi(f[0]);
                r.vehicleId = stoi(f[1]);
                r.vehicleName = f[2];
                r.customerName = f[3];
                r.phone = f[4];
                r.license = f[5];
                r.bookedHours = stoi(f[6]);
                r.rent = stod(f[7]);
                r.deposit = stod(f[8]);
                r.bookingTime = f[9];

                Vehicle* v = findVehicle(r.vehicleId);

                if (v) {
                    // Prevent duplicate active records for one vehicle.
                    bool alreadyLoaded = false;

                    for (const auto& existing : activeRentals) {
                        if (existing.rentalId == r.rentalId ||
                            existing.vehicleId == r.vehicleId) {
                            alreadyLoaded = true;
                            break;
                        }
                    }

                    if (!alreadyLoaded) {
                        v->available = false;
                        activeRentals.push_back(r);
                    }
                }
            } catch (...) {
                cout << "Warning: An invalid active rental record was skipped.\n";
            }
        }
    }

    bool saveActiveRentals() {
        ofstream out(ACTIVE_FILE);

        if (!out) {
            cout << "Error: Could not save active rentals.\n";
            return false;
        }

        for (const auto& r : activeRentals) {
            out << r.rentalId << '|'
                << r.vehicleId << '|'
                << r.vehicleName << '|'
                << r.customerName << '|'
                << r.phone << '|'
                << r.license << '|'
                << r.bookedHours << '|'
                << r.rent << '|'
                << r.deposit << '|'
                << r.bookingTime << '\n';
        }

        return static_cast<bool>(out);
    }

    void showAvailableVehicles() {
        title("AVAILABLE VEHICLES");

        cout << left
             << setw(6) << "ID"
             << setw(31) << "Vehicle"
             << setw(14) << "Category"
             << setw(8) << "Seats"
             << setw(14) << "Rs./Hour"
             << setw(12) << "Status" << '\n';

        line('-');

        bool found = false;

        for (const auto& v : vehicles) {
            if (v.available) {
                v.display();
                found = true;
            }
        }

        if (!found)
            cout << "No vehicles are currently available.\n";
    }

    void printInvoice(const Rental& r) {
        title("SMART VEHICLE RENTAL - INVOICE");

        cout << "Rental ID           : " << r.rentalId << '\n';
        cout << "Booking time        : " << r.bookingTime << '\n';
        cout << "Customer            : " << r.customerName << '\n';
        cout << "Mobile              : " << r.phone << '\n';
        cout << "Driving licence     : " << r.license << '\n';

        line('-');

        cout << "Vehicle             : " << r.vehicleName << '\n';
        cout << "Rental duration     : " << r.bookedHours << " hours\n";
        cout << "Rental charge       : " << money(r.rent) << '\n';
        cout << "Refundable deposit  : " << money(r.deposit) << '\n';

        line('-');

        cout << "TOTAL PAYABLE NOW   : "
             << money(r.rent + r.deposit) << '\n';

        cout << "\nDeposit is refundable after return inspection "
                "and applicable deductions.\n";

        line();

        ofstream out(INVOICE_FILE, ios::app);

        if (!out) {
            cout << "Warning: Invoice file could not be opened.\n";
            return;
        }

        out << "\n========================================\n";
        out << "SMART VEHICLE RENTAL - INVOICE\n";
        out << "Rental ID: " << r.rentalId << '\n';
        out << "Customer: " << r.customerName << '\n';
        out << "Mobile: " << r.phone << '\n';
        out << "Vehicle: " << r.vehicleName << '\n';
        out << "Booking time: " << r.bookingTime << '\n';
        out << "Duration: " << r.bookedHours << " hours\n";
        out << "Rent: " << money(r.rent) << '\n';
        out << "Deposit: " << money(r.deposit) << '\n';
        out << "Total payable: " << money(r.rent + r.deposit) << '\n';
        out << "========================================\n";
    }

    void bookVehicle() {
        title("NEW VEHICLE BOOKING");
        showAvailableVehicles();

        bool anyAvailable = false;

        for (const auto& v : vehicles)
            if (v.available)
                anyAvailable = true;

        if (!anyAvailable)
            return;

        int id = readInt(
            "\nEnter vehicle ID to book (0 to cancel): ", 0, 999999);

        if (id == 0)
            return;

        Vehicle* v = findVehicle(id);

        if (!v) {
            cout << "Vehicle ID not found.\n";
            return;
        }

        if (!v->available) {
            cout << "This vehicle is already rented.\n";
            return;
        }

        Rental r;
        r.rentalId = nextRentalId;
        r.vehicleId = v->id;
        r.vehicleName = v->name;

        r.customerName = readName("Customer full name: ");
        r.phone = readPhone("Mobile number (10 digits): ");
        r.license = readLicense("Driving licence number: ");

        r.bookedHours = readInt(
            "Rental duration in hours (1-720): ", 1, 720);

        r.rent = v->calculateRent(r.bookedHours);
        r.deposit = v->deposit;
        r.bookingTime = currentTime();

        activeRentals.push_back(r);
        v->available = false;

        if (!saveActiveRentals()) {
            activeRentals.pop_back();
            v->available = true;
            cout << "Booking was not saved. Please try again.\n";
            return;
        }

        ++nextRentalId;
        printInvoice(r);

        cout << "\nBooking confirmed. Invoice generated automatically.\n";
    }

    void showActiveRentals() {
        title("ACTIVE RENTALS");

        if (activeRentals.empty()) {
            cout << "No active rentals.\n";
            return;
        }

        for (const auto& r : activeRentals) {
            cout << "\nRental ID: " << r.rentalId
                 << "\nCustomer: " << r.customerName
                 << "\nMobile: " << r.phone
                 << "\nVehicle: " << r.vehicleName
                 << "\nDuration: " << r.bookedHours << " hours"
                 << "\nRent: " << money(r.rent)
                 << "\nDeposit: " << money(r.deposit) << '\n';

            line('-');
        }
    }

    void returnVehicle() {
        title("VEHICLE RETURN & FINAL BILL");

        if (activeRentals.empty()) {
            cout << "There are no active rentals to return.\n";
            return;
        }

        showActiveRentals();

        int id = readInt(
            "Enter rental ID to return (0 to cancel): ", 0, 999999);

        if (id == 0)
            return;

        auto it = find_if(activeRentals.begin(), activeRentals.end(),
            [id](const Rental& r) {
                return r.rentalId == id;
            });

        if (it == activeRentals.end()) {
            cout << "Active rental ID not found.\n";
            return;
        }

        Rental r = *it;
        Vehicle* v = findVehicle(r.vehicleId);

        int actualHours = readInt(
            "Actual rental duration in hours (1-2000): ", 1, 2000);

        double actualRent = v
            ? v->calculateRent(actualHours)
            : r.rent;

        double lateFee = 0;

        if (actualHours > r.bookedHours) {
            int extraHours = actualHours - r.bookedHours;
            double rate = v ? v->hourlyRate : 0;
            lateFee = extraHours * rate * 1.25;
        }

        double damage = readDouble(
            "Damage/other deductions (Rs., enter 0 if none): ",
            0, 10000000);

        double finalCharges = actualRent + lateFee + damage;

        // Apply deductions to the refundable deposit.
        double depositUsed = min(r.deposit, lateFee + damage);
        double depositRefund = r.deposit - depositUsed;

        // Original rental amount was already payable at booking.
        double additionalBalance = max(0.0, finalCharges - r.rent);
        double unusedRentalBalance = max(0.0, r.rent - finalCharges);

        title("FINAL RETURN BILL");

        cout << "Rental ID               : " << r.rentalId << '\n';
        cout << "Customer                : " << r.customerName << '\n';
        cout << "Vehicle                 : " << r.vehicleName << '\n';
        cout << "Booked rent             : " << money(r.rent) << '\n';
        cout << "Actual rental charge    : " << money(actualRent) << '\n';
        cout << "Late return fee         : " << money(lateFee) << '\n';
        cout << "Damage/other deductions : " << money(damage) << '\n';
        cout << "Total final charges     : " << money(finalCharges) << '\n';
        cout << "Additional balance due  : "
             << money(additionalBalance) << '\n';
        cout << "Deposit refund          : " << money(depositRefund) << '\n';

        if (unusedRentalBalance > 0)
            cout << "Unused rental balance   : "
                 << money(unusedRentalBalance) << '\n';

        line();

        // Save completed rental to history before removing it.
        ofstream history(HISTORY_FILE, ios::app);

        if (!history) {
            cout << "Error: Could not save rental history. "
                    "Return was not finalized.\n";
            return;
        }

        history << r.rentalId << '|'
                << r.customerName << '|'
                << r.vehicleName << '|'
                << r.rent << '|'
                << actualRent << '|'
                << lateFee << '|'
                << damage << '|'
                << currentTime() << '\n';

        history.close();

        if (!history) {
            cout << "Error writing rental history. "
                    "Return was not finalized.\n";
            return;
        }

        ofstream inv(INVOICE_FILE, ios::app);

        if (inv) {
            inv << "\n========================================\n";
            inv << "VEHICLE RETURN BILL\n";
            inv << "Rental ID: " << r.rentalId << '\n';
            inv << "Customer: " << r.customerName << '\n';
            inv << "Vehicle: " << r.vehicleName << '\n';
            inv << "Actual rent: " << money(actualRent) << '\n';
            inv << "Late fee: " << money(lateFee) << '\n';
            inv << "Damage/other deductions: " << money(damage) << '\n';
            inv << "Final charges: " << money(finalCharges) << '\n';
            inv << "Additional balance due: "
                << money(additionalBalance) << '\n';
            inv << "Deposit refund: " << money(depositRefund) << '\n';
            inv << "========================================\n";
        }

        size_t index = static_cast<size_t>(
            distance(activeRentals.begin(), it));

        activeRentals.erase(activeRentals.begin() + index);

        if (v)
            v->available = true;

        if (!saveActiveRentals()) {
            cout << "Warning: Could not update the active rentals file.\n";
        }

        cout << "\nVehicle returned successfully. Final bill generated.\n";
    }

    void showReports() {
        title("RENTAL HISTORY & REVENUE REPORT");

        ifstream in(HISTORY_FILE);

        if (!in) {
            cout << "No rental history file found yet.\n";
            return;
        }

        string row;
        int count = 0;
        double totalRent = 0;
        double totalLateFees = 0;
        double totalDeductions = 0;

        while (getline(in, row)) {
            vector<string> f = splitRecord(row);

            if (f.size() != 8)
                continue;

            try {
                int rentalId = stoi(f[0]);
                double actualRent = stod(f[4]);
                double lateFee = stod(f[5]);
                double damage = stod(f[6]);

                ++count;
                totalRent += actualRent;
                totalLateFees += lateFee;
                totalDeductions += damage;

                cout << "Rental #" << rentalId
                     << " | " << f[1]
                     << " | " << f[2]
                     << " | Actual rent: " << money(actualRent)
                     << '\n';
            } catch (...) {
                cout << "Warning: Skipped an invalid history record.\n";
            }
        }

        line('-');

        cout << "Completed rentals : " << count << '\n';
        cout << "Recorded rent     : " << money(totalRent) << '\n';
        cout << "Late fees         : " << money(totalLateFees) << '\n';
        cout << "Other deductions  : " << money(totalDeductions) << '\n';
        cout << "Active rentals    : " << activeRentals.size() << '\n';

        cout << "\nReport is calculated from rental_history.txt.\n";
        cout << "These figures are recorded charges, not audited profit.\n";
    }

public:
    SmartRentalSystem() {
        loadVehicles();
        loadActiveRentals();
        loadNextRentalId();
    }

    void run() {
        while (true) {
            title("SMART VEHICLE RENTAL SYSTEM");

            cout << "1. Book a vehicle (instant invoice)\n"
                 << "2. View active rentals\n"
                 << "3. Return vehicle (automatic final bill)\n"
                 << "4. Rental history and reports\n"
                 << "0. Exit\n";

            int choice = readInt("\nEnter your choice: ", 0, 4);

            switch (choice) {
                case 1:
                    bookVehicle();
                    break;

                case 2:
                    showActiveRentals();
                    break;

                case 3:
                    returnVehicle();
                    break;

                case 4:
                    showReports();
                    break;

                case 0:
                    cout << "\nThank you for using Smart Vehicle Rental!\n";
                    return;
            }

            cout << "\nPress Enter to continue...";
            string dummy;
            getline(cin, dummy);
        }
    }
};

int main() {
    SmartRentalSystem system;
    system.run();
    return 0;
}