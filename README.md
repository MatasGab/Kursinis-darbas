
from abc import ABC, abstractmethod
from datetime import datetime
import yfinance as yf
import sqlite3

class Logger:
    def log(self, message: str):
        print(f"[LOG - {datetime.now()}]: {message}")

class DatabaseManager:
    def __init__(self, db_file='finance.db'):
        self.conn = sqlite3.connect(db_file)
        self.create_tables()

  def create_tables(self):
        with self.conn:
            self.conn.execute("""CREATE TABLE IF NOT EXISTS transactions (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                amount REAL,
                category TEXT,
                date TEXT,
                description TEXT
            )""")

  def add_transaction(self, amount, category, date, description):
        with self.conn:
            self.conn.execute("""INSERT INTO transactions (amount, category, date, description)
                VALUES (?, ?, ?, ?)""", (amount, category, date, description))

  def get_transactions(self):
        cursor = self.conn.cursor()
        cursor.execute('SELECT amount, category, date, description FROM transactions')
        return cursor.fetchall()

class Transaction:
    def __init__(self, amount: float, category: str, date: datetime, description: str):
        self._amount = amount
        self._category = category
        self._date = date
        self._description = description

  def __str__(self):
        return f"{self._date.date()} | {self._category}: ${self._amount:.2f} - {self._description}"

class Account:
    def __init__(self, name: str):
        self._name = name
        self._transactions = []

  def add_transaction(self, transaction: Transaction):
        self._transactions.append(transaction)

  def get_balance(self):
        return sum(t._amount for t in self._transactions)

  def list_transactions(self):
        return [str(t) for t in self._transactions]

class Company:
    def __init__(self, symbol: str):
        self.symbol = symbol
        self.stock_data = None

  def update_stock_data(self):
        ticker = yf.Ticker(self.symbol)
        self.stock_data = ticker.history(period="1d")

  def current_price(self):
        if self.stock_data is not None and not self.stock_data.empty:
            return self.stock_data["Close"].iloc[-1]
        return None

class Portfolio:
    def __init__(self):
        self._companies = []

  def add_company(self, company: Company):
        self._companies.append(company)

  def get_company_prices(self):
        prices = {}
        for c in self._companies:
            c.update_stock_data()
            prices[c.symbol] = c.current_price()
        return prices

class ReportStrategy(ABC):
    @abstractmethod
    def generate(self, account: Account):
        pass

class SimpleTextReport(ReportStrategy):
    def generate(self, account: Account):
        print("--- Simple Text Report ---")
        print("Account:", account._name)
        for t in account.list_transactions():
            print(t)
        print("Balance:", account.get_balance())

class ReportFactory:
    @staticmethod
    def get_report(format_type: str) -> ReportStrategy:
        if format_type == "text":
            return SimpleTextReport()
        raise ValueError("Unknown report format")

class FinanceTrackerApp:
    def __init__(self, logger: Logger, db: DatabaseManager):
        self.logger = logger
        self.db = db
        self.account = Account("Personal Account")
        self.portfolio = Portfolio()
        self.load_transactions()

  def add_transaction(self, amount: float, category: str, description: str):
        transaction = Transaction(amount, category, datetime.now(), description)
        self.account.add_transaction(transaction)
        self.db.add_transaction(amount, category, transaction._date.isoformat(), description)
        self.logger.log(f"Transaction added: {transaction}")

  def load_transactions(self):
        for amount, category, date, description in self.db.get_transactions():
            t = Transaction(float(amount), category, datetime.fromisoformat(date), description)
            self.account.add_transaction(t)

  def track_company(self, symbol: str):
        company = Company(symbol)
        self.portfolio.add_company(company)
        self.logger.log(f"Tracking new company: {symbol}")

  def show_company_prices(self):
        prices = self.portfolio.get_company_prices()
        for symbol, price in prices.items():
            if price is not None:
                print(f"{symbol}: ${price:.2f}")
            else:
                print(f"{symbol}: Price not available")

  def generate_report(self, format_type: str):
        report = ReportFactory.get_report(format_type)
        report.generate(self.account)

if __name__ == "__main__":
    logger = Logger()
    db = DatabaseManager()
    app = FinanceTrackerApp(logger, db)
    app.add_transaction(1000, "Salary", "April Salary")
    app.add_transaction(-150, "Groceries", "Weekly groceries")
    app.track_company("TSLA")
    app.track_company("AAPL")
    app.show_company_prices()
    app.generate_report("text")
