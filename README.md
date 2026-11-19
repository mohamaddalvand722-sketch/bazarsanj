# bazarsanj
Sales Management and Data Analysis System built with Python, SQLite, Tkinter, Pandas and NumPy.

import tkinter as tk
from tkinter import messagebox, filedialog
import sqlite3
import csv
import shutil
from datetime import datetime

# ================= DATABASE =================

db = sqlite3.connect("bazarsanj_ultra.db")

db.executescript("""
CREATE TABLE IF NOT EXISTS products(
 id INTEGER PRIMARY KEY AUTOINCREMENT,
 name TEXT NOT NULL,
 price REAL NOT NULL,
 cost REAL NOT NULL,
 stock INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS sales(
 id INTEGER PRIMARY KEY AUTOINCREMENT,
 product_id INTEGER,
 name TEXT NOT NULL,
 qty INTEGER NOT NULL,
 total REAL NOT NULL,
 profit REAL NOT NULL,
 date TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS expenses(
 id INTEGER PRIMARY KEY AUTOINCREMENT,
 title TEXT NOT NULL,
 amount REAL NOT NULL,
 date TEXT NOT NULL
);
""")
db.commit()

# ================= THEME =================

BG = "#171716"
PANEL = "#292724"
CARD = "#35312C"
ORANGE = "#D98243"
COPPER = "#B86F42"
WHITE = "#F5F0E8"
MUTED = "#B9AEA2"
RED = "#D65C50"
GREEN = "#4BA77A"

root = tk.Tk()
root.title("BAZARSANJ ULTRA")
root.geometry("390x700")
root.configure(bg=BG)

# ================= HELPERS =================

def money(value):
    return f"{value:,.0f}"

def now():
    return datetime.now().strftime("%Y-%m-%d %H:%M")

def clear():
    for w in root.winfo_children():
        w.destroy()

def button(parent, text, command, color=ORANGE):
    tk.Button(
        parent,
        text=text,
        command=command,
        bg=color,
        fg=WHITE,
        activebackground=COPPER,
        activeforeground=WHITE,
        font=("Arial", 11, "bold"),
        relief="flat",
        bd=0,
        height=2
    ).pack(fill="x", padx=12, pady=4)

def label(parent, text, size=11, color=WHITE):
    tk.Label(
        parent,
        text=text,
        bg=BG,
        fg=color,
        font=("Arial", size, "bold" if size >= 14 else "normal"),
        justify="left",
        wraplength=330
    ).pack(anchor="w", padx=14, pady=5)

def entry(parent, title):
    tk.Label(
        parent,
        text=title,
        bg=BG,
        fg=MUTED,
        font=("Arial", 10)
    ).pack(anchor="w", padx=14, pady=(7, 1))

    e = tk.Entry(
        parent,
        bg=CARD,
        fg=WHITE,
        insertbackground=WHITE,
        font=("Arial", 12),
        relief="flat"
    )
    e.pack(fill="x", padx=12, pady=3, ipady=8)
    return e

def page(title):
    clear()

    tk.Label(
        root,
        text="BAZARSANJ ULTRA",
        bg=BG,
        fg=ORANGE,
        font=("Arial", 18, "bold")
    ).pack(pady=(10, 2))

    tk.Label(
        root,
        text=title,
        bg=BG,
        fg=MUTED,
        font=("Arial", 10)
    ).pack(pady=(0, 8))

    area = tk.Frame(root, bg=BG)
    area.pack(fill="both", expand=True)

    canvas = tk.Canvas(
        area,
        bg=BG,
        highlightthickness=0
    )

    scroll = tk.Scrollbar(
        area,
        orient="vertical",
        command=canvas.yview
    )

    content = tk.Frame(canvas, bg=BG)

    window_id = canvas.create_window(
        (0, 0),
        window=content,
        anchor="nw"
    )

    content.bind(
        "<Configure>",
        lambda e: canvas.configure(
            scrollregion=canvas.bbox("all")
        )
    )

    canvas.bind(
        "<Configure>",
        lambda e: canvas.itemconfig(
            window_id,
            width=e.width
        )
    )

    canvas.configure(yscrollcommand=scroll.set)

    canvas.pack(side="left", fill="both", expand=True)
    scroll.pack(side="right", fill="y")

    return content

def card(parent, title, value, color=ORANGE):
    box = tk.Frame(parent, bg=PANEL)
    box.pack(fill="x", padx=12, pady=5)

    tk.Label(
        box,
        text=title,
        bg=PANEL,
        fg=MUTED,
        font=("Arial", 10)
    ).pack(anchor="w", padx=12, pady=(8, 2))

    tk.Label(
        box,
        text=value,
        bg=PANEL,
        fg=color,
        font=("Arial", 18, "bold")
    ).pack(anchor="w", padx=12, pady=(2, 10))

# ================= DASHBOARD =================

def home():
    p = page("MANAGEMENT DASHBOARD")

    revenue = db.execute(
        "SELECT COALESCE(SUM(total),0) FROM sales"
    ).fetchone()[0]

    profit = db.execute(
        "SELECT COALESCE(SUM(profit),0) FROM sales"
    ).fetchone()[0]

    expenses = db.execute(
        "SELECT COALESCE(SUM(amount),0) FROM expenses"
    ).fetchone()[0]

    products = db.execute(
        "SELECT COUNT(*) FROM products"
    ).fetchone()[0]

    stock = db.execute(
        "SELECT COALESCE(SUM(stock),0) FROM products"
    ).fetchone()[0]

    low = db.execute(
        "SELECT COUNT(*) FROM products WHERE stock<=5"
    ).fetchone()[0]

    card(p, "TOTAL SALES", money(revenue))
    card(p, "GROSS PROFIT", money(profit), GREEN)
    card(p, "EXPENSES", money(expenses), RED)
    card(p, "NET PROFIT", money(profit-expenses), GREEN)
    card(p, "PRODUCTS", str(products))
    card(p, "TOTAL STOCK", str(stock))

    if low:
        label(p, f"LOW STOCK ALERT: {low} products", 12, RED)

    button(p, "PRODUCT MANAGEMENT", products_page)
    button(p, "NEW SALE", sales_page)
    button(p, "ADD STOCK", stock_page)
    button(p, "EXPENSE MANAGEMENT", expenses_page)
    button(p, "REPORTS & EXPORT", reports_page)
    button(p, "DATABASE BACKUP", backup)

# ================= PRODUCTS =================

def products_page():
    p = page("PRODUCT MANAGEMENT")

    name = entry(p, "Product name")
    price = entry(p, "Selling price")
    cost = entry(p, "Purchase cost")
    stock = entry(p, "Stock quantity")

    selected = {"id": None}

    def save():
        try:
            n = name.get().strip()
            pr = float(price.get())
            co = float(cost.get())
            st = int(stock.get())

            if not n or min(pr, co, st) < 0:
                raise ValueError

            if selected["id"] is None:
                db.execute(
                    "INSERT INTO products(name,price,cost,stock) VALUES(?,?,?,?)",
                    (n, pr, co, st)
                )
            else:
                db.execute(
                    "UPDATE products SET name=?,price=?,cost=?,stock=? WHERE id=?",
                    (n, pr, co, st, selected["id"])
                )

            db.commit()
            messagebox.showinfo("Success", "Product saved")
            products_page()

        except ValueError:
            messagebox.showerror("Error", "Check product information")

    button(p, "SAVE PRODUCT", save)

    listing = tk.Frame(p, bg=BG)
    listing.pack(fill="x", pady=8)

    rows = db.execute(
        "SELECT id,name,price,cost,stock FROM products ORDER BY id DESC"
    ).fetchall()

    for pid, n, pr, co, st in rows:
        box = tk.Frame(listing, bg=PANEL)
        box.pack(fill="x", padx=12, pady=4)

        tk.Label(
            box,
            text=f"{n}\nPrice: {money(pr)} | Cost: {money(co)}\nStock: {st}",
            bg=PANEL,
            fg=WHITE,
            font=("Arial", 10),
            justify="left"
        ).pack(anchor="w", padx=9, pady=7)

        def edit(pid=pid, n=n, pr=pr, co=co, st=st):
            selected["id"] = pid

            for e, value in [
                (name, n),
                (price, pr),
                (cost, co),
                (stock, st)
            ]:
                e.delete(0, "end")
                e.insert(0, str(value))

        def delete(pid=pid):
            used = db.execute(
                "SELECT COUNT(*) FROM sales WHERE product_id=?",
                (pid,)
            ).fetchone()[0]

            if used:
                messagebox.showwarning(
                    "Warning",
                    "This product has sales history and cannot be deleted."
                )
                return

            if messagebox.askyesno("Delete", "Delete product?"):
                db.execute("DELETE FROM products WHERE id=?", (pid,))
                db.commit()
                products_page()

        actions = tk.Frame(box, bg=PANEL)
        actions.pack(fill="x", padx=5, pady=5)

        tk.Button(
            actions, text="EDIT",
            command=edit, bg=ORANGE, fg=WHITE
        ).pack(side="left", expand=True, fill="x", padx=3)

        tk.Button(
            actions, text="DELETE",
            command=delete, bg=RED, fg=WHITE
        ).pack(side="left", expand=True, fill="x", padx=3)

    button(p, "BACK", home)

# ================= SALES =================

def sales_page():
    p = page("NEW SALE")

    rows = db.execute(
        "SELECT id,name,price,stock FROM products WHERE stock>0"
    ).fetchall()

    if not rows:
        label(p, "No products available", 14, RED)

    for pid, name, price, stock in rows:
        box = tk.Frame(p, bg=PANEL)
        box.pack(fill="x", padx=12, pady=5)

        tk.Label(
            box,
            text=f"{name}\nPrice: {money(price)} | Stock: {stock}",
            bg=PANEL,
            fg=WHITE,
            font=("Arial", 11),
            justify="left"
        ).pack(anchor="w", padx=10, pady=6)

        def sell(pid=pid):
            sale_popup(pid)

        tk.Button(
            box,
            text="SELL",
            command=sell,
            bg=ORANGE,
            fg=WHITE
        ).pack(fill="x", padx=8, pady=5)

    button(p, "BACK", home)

def sale_popup(pid):
    win = tk.Toplevel(root)
    win.title("New Sale")
    win.geometry("300x260")
    win.configure(bg=BG)

    tk.Label(
        win,
        text="SALE QUANTITY",
        bg=BG,
        fg=ORANGE,
        font=("Arial", 14, "bold")
    ).pack(pady=15)

    qty = tk.Entry(
        win,
        bg=CARD,
        fg=WHITE,
        insertbackground=WHITE,
        font=("Arial", 14)
    )
    qty.pack(fill="x", padx=20, pady=10)

    def confirm():
        try:
            q = int(qty.get())

            if q <= 0:
                raise ValueError

            with db:
                product = db.execute(
                    "SELECT name,price,cost,stock FROM products WHERE id=?",
                    (pid,)
                ).fetchone()

                if not product or q > product[3]:
                    raise ValueError

                name, price, cost, available = product
                total = price * q
                profit = (price-cost) * q

                db.execute(
                    """INSERT INTO sales
                    (product_id,name,qty,total,profit,date)
                    VALUES(?,?,?,?,?,?)""",
                    (pid, name, q, total, profit, now())
                )

                db.execute(
                    "UPDATE products SET stock=stock-? WHERE id=?",
                    (q, pid)
                )

            win.destroy()
            messagebox.showinfo(
                "Sale completed",
                f"Total: {money(total)}\nProfit: {money(profit)}"
            )
            sales_page()

        except (ValueError, TypeError):
            messagebox.showerror("Error", "Invalid quantity or low stock")

    tk.Button(
        win,
        text="CONFIRM SALE",
        command=confirm,
        bg=ORANGE,
        fg=WHITE,
        font=("Arial", 12, "bold")
    ).pack(fill="x", padx=20, pady=12)

# ================= ADD STOCK =================

def stock_page():
    p = page("INVENTORY ENTRY")

    rows = db.execute(
        "SELECT id,name,stock FROM products ORDER BY name"
    ).fetchall()

    if not rows:
        label(p, "Add products first", 13, RED)

    for pid, name, stock in rows:
        box = tk.Frame(p, bg=PANEL)
        box.pack(fill="x", padx=12, pady=5)

        tk.Label(
            box,
            text=f"{name} | Current stock: {stock}",
            bg=PANEL,
            fg=WHITE,
            font=("Arial", 10)
        ).pack(anchor="w", padx=8, pady=7)

        def add(pid=pid, name=name):
            win = tk.Toplevel(root)
            win.title("Add stock")
            win.geometry("290x230")
            win.configure(bg=BG)

            tk.Label(
                win, text=name,
                bg=BG, fg=ORANGE,
                font=("Arial", 13, "bold")
            ).pack(pady=10)

            q = tk.Entry(win, font=("Arial", 13))
            q.pack(padx=20, pady=10)

            def save():
                try:
                    amount = int(q.get())
                    if amount <= 0:
                        raise ValueError

                    db.execute(
                        "UPDATE products SET stock=stock+? WHERE id=?",
                        (amount, pid)
                    )
                    db.commit()
                    win.destroy()
                    stock_page()

                except ValueError:
                    messagebox.showerror("Error", "Invalid quantity")

            tk.Button(
                win, text="ADD STOCK",
                command=save, bg=ORANGE, fg=WHITE
            ).pack(pady=10)

        tk.Button(
            box, text="ADD",
            command=add,
            bg=GREEN, fg=WHITE
        ).pack(fill="x", padx=8, pady=5)

    button(p, "BACK", home)

# ================= EXPENSES =================

def expenses_page():
    p = page("EXPENSE MANAGEMENT")

    title = entry(p, "Expense title")
    amount = entry(p, "Expense amount")

    def save():
        try:
            t = title.get().strip()
            a = float(amount.get())

            if not t or a <= 0:
                raise ValueError

            db.execute(
                "INSERT INTO expenses(title,amount,date) VALUES(?,?,?)",
                (t, a, now())
            )
            db.commit()

            messagebox.showinfo("Success", "Expense saved")
            expenses_page()

        except ValueError:
            messagebox.showerror("Error", "Invalid expense")

    button(p, "SAVE EXPENSE", save)

    rows = db.execute(
        "SELECT title,amount,date FROM expenses ORDER BY id DESC"
    ).fetchall()

    for title, amount, date in rows:
        label(
            p,
            f"{title} | {money(amount)}\n{date}",
            10,
            MUTED
        )

    button(p, "BACK", home)

# ================= REPORTS =================

def reports_page():
    p = page("REPORTS & ANALYTICS")

    revenue = db.execute(
        "SELECT COALESCE(SUM(total),0) FROM sales"
    ).fetchone()[0]

    profit = db.execute(
        "SELECT COALESCE(SUM(profit),0) FROM sales"
    ).fetchone()[0]

    expenses = db.execute(
        "SELECT COALESCE(SUM(amount),0) FROM expenses"
    ).fetchone()[0]

    sold = db.execute(
        "SELECT COALESCE(SUM(qty),0) FROM sales"
    ).fetchone()[0]

    card(p, "TOTAL REVENUE", money(revenue))
    card(p, "GROSS PROFIT", money(profit), GREEN)
    card(p, "EXPENSES", money(expenses), RED)
    card(p, "NET PROFIT", money(profit-expenses), GREEN)
    card(p, "UNITS SOLD", str(sold))

    rows = db.execute(
        "SELECT name,qty,total,profit,date FROM sales ORDER BY id DESC LIMIT 50"
    ).fetchall()

    label(p, "RECENT SALES", 14, ORANGE)

    for name, qty, total, profit, date in rows:
        label(
            p,
            f"{name} | Qty: {qty}\n"
            f"Total: {money(total)} | Profit: {money(profit)}\n{date}",
            10
        )

    button(p, "EXPORT SALES CSV", export_sales)
    button(p, "EXPORT PRODUCTS CSV", export_products)
    button(p, "BACK", home)

# ================= EXPORT =================

def export_file(filename, headers, rows):
    path = filedialog.asksaveasfilename(
        defaultextension=".csv",
        initialfile=filename,
        filetypes=[("CSV files", "*.csv")]
    )

    if not path:
        return

    try:
        with open(path, "w", newline="", encoding="utf-8-sig") as f:
            writer = csv.writer(f)
            writer.writerow(headers)
            writer.writerows(rows)

        messagebox.showinfo("Export", "File saved successfully")

    except Exception as e:
        messagebox.showerror("Export error", str(e))

def export_sales():
    rows = db.execute(
        "SELECT id,name,qty,total,profit,date FROM sales"
    ).fetchall()

    export_file(
        "ultra_sales.csv",
        ["ID", "Product", "Quantity", "Total", "Profit", "Date"],
        rows
    )

def export_products():
    rows = db.execute(
        "SELECT id,name,price,cost,stock FROM products"
    ).fetchall()

    export_file(
        "ultra_products.csv",
        ["ID", "Name", "Price", "Cost", "Stock"],
        rows
    )

# ================= BACKUP =================

def backup():
    path = filedialog.asksaveasfilename(
        defaultextension=".db",
        initialfile="bazarsanj_backup.db",
        filetypes=[("Database", "*.db")]
    )

    if not path:
        return

    try:
        db.commit()
        shutil.copy2("bazarsanj_ultra.db", path)
        messagebox.showinfo("Backup", "Backup completed")

    except Exception as e:
        messagebox.showerror("Backup error", str(e))

# ================= START =================

def close_app():
    db.commit()
    db.close()
    root.destroy()

root.protocol("WM_DELETE_WINDOW", close_app)

home()
root.mainloop()
