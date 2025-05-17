
import sqlite3
from datetime import datetime
from kivy.app import App
from kivy.core.window import Window
from kivy.uix.screenmanager import ScreenManager, Screen
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.label import Label
from kivy.uix.textinput import TextInput
from kivy.uix.button import Button
from kivy.uix.scrollview import ScrollView
from kivy.uix.gridlayout import GridLayout
from kivy.uix.popup import Popup
from kivy.uix.spinner import Spinner
from kivy.uix.togglebutton import ToggleButton
from kivy.utils import get_color_from_hex
from kivy.metrics import dp

# ======================
# STYLE CONFIGURATION
# ======================
Window.clearcolor = get_color_from_hex('#f5f5f5')  # Light gray background

class EnhancedButton(Button):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.background_color = get_color_from_hex('#4CAF50')  # Green
        self.color = get_color_from_hex('#FFFFFF')  # White text
        self.font_size = '16sp'
        self.bold = True
        self.size_hint_y = None
        self.height = dp(40)

class HeaderLabel(Label):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.color = get_color_from_hex('#2c3e50')  # Dark blue
        self.font_size = '20sp'
        self.bold = True

class SecondaryLabel(Label):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.color = get_color_from_hex('#34495e')  # Dark gray-blue
        self.font_size = '14sp'

# ======================
# DATABASE SETUP
# ======================
def connect_db():
    return sqlite3.connect('pharmacy_inventory.db')

def create_tables():
    with connect_db() as conn:
        cursor = conn.cursor()
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS products (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                category TEXT,
                quantity_remaining INTEGER,
                quantity_to_expire INTEGER,
                price REAL,
                expiry_date TEXT,
                mode_of_payment TEXT
            )
        ''')
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS sales (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                product_id INTEGER,
                quantity_sold INTEGER,
                sale_date TEXT,
                FOREIGN KEY(product_id) REFERENCES products(id)
            )
        ''')
        conn.commit()

create_tables()
# ======================
# UTILITIES
# ======================
def validate_date(date_text):
    try:
        datetime.strptime(date_text, '%Y-%m-%d')
        return True
    except ValueError:
        return False

def show_popup(title, message, size=(0.6, 0.3)):
    Popup(
        title=title,
        content=Label(text=message),
        size_hint=size
    ).open()

# ======================
# SCREENS
# ======================
class MenuScreen(Screen):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        layout = BoxLayout(orientation='vertical', padding=20, spacing=20)
        layout.add_widget(HeaderLabel(text="Pharmacy Inventory System"))

        button_layout = BoxLayout(orientation='vertical', spacing=10)
        button_layout.add_widget(EnhancedButton(
            text='Add Product',
            on_press=lambda x: setattr(self.manager, 'current', 'add_product')
        ))
        button_layout.add_widget(EnhancedButton(
            text='View Products',
            on_press=lambda x: setattr(self.manager, 'current', 'view_products')
        ))
        button_layout.add_widget(EnhancedButton(
            text='Record Sale',
            on_press=lambda x: setattr(self.manager, 'current', 'record_sale')
        ))
        button_layout.add_widget(EnhancedButton(
            text='Expiry Alerts (7 days)',
            on_press=lambda x: setattr(self.manager, 'current', 'expiry_reminder')
        ))
        button_layout.add_widget(EnhancedButton(
            text='Low Stock Alerts',
            on_press=lambda x: setattr(self.manager, 'current', 'low_stock_alerts')
        ))

        layout.add_widget(button_layout)
        self.add_widget(layout)

class AddProductScreen(Screen):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.fields = {}
        self.layout = BoxLayout(orientation='vertical', padding=10, spacing=10)
        
        labels = [
            "Drug Name", "Category", "Qty Remaining", "Qty to Expire",
            "Price (MWK)", "Expiry Date", "Mode of Payment"
        ]
        
        for label_text in labels:
            box = BoxLayout(size_hint_y=None, height=dp(40))
            label = SecondaryLabel(text=label_text, size_hint_x=0.4)
            text_input = TextInput(multiline=False)
            self.fields[label_text] = text_input
            box.add_widget(label)
            box.add_widget(text_input)
            self.layout.add_widget(box)
        
        self.layout.add_widget(EnhancedButton(
            text="Save Product", 
            on_press=self.save_product
        ))
        self.layout.add_widget(EnhancedButton(
            text="Back",
            on_press=lambda x: setattr(self.manager, 'current', 'menu')
        ))
        self.add_widget(self.layout)

    def save_product(self, instance):
        try:
            with connect_db() as conn:
                cursor = conn.cursor()
                if not all(field.text.strip() for field in self.fields.values()):
                    raise ValueError("All fields are required")

                data = (
                    self.fields["Drug Name"].text.strip(),
                    self.fields["Category"].text.strip(),
                    int(self.fields["Qty Remaining"].text),
                    int(self.fields["Qty to Expire"].text),
                    float(self.fields["Price (MWK)"].text),
                    self.fields["Expiry Date"].text.strip(),
                    self.fields["Mode of Payment"].text.strip()
                )

                if not validate_date(data[5]):
                    raise ValueError("Invalid date format. Use YYYY-MM-DD")

                cursor.execute('''
                    INSERT INTO products 
                    (name, category, quantity_remaining, quantity_to_expire, 
                     price, expiry_date, mode_of_payment)
                    VALUES (?, ?, ?, ?, ?, ?, ?)
                ''', data)
                conn.commit()

                show_popup('Success', 'Product saved successfully')
                for field in self.fields.values():
                    field.text = ''

        except Exception as e:
            show_popup('Error', str(e), size=(0.7, 0.3))

class ViewProductsScreen(Screen):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.layout = BoxLayout(orientation='vertical', padding=10, spacing=10)
        
        # Filter Controls
        filter_layout = BoxLayout(size_hint_y=None, height=dp(40), spacing=5)
        self.payment_filter = Spinner(
            text='All Payment Modes',
            values=['All', 'Cash', 'Insurance'],
            size_hint_x=0.4
        )
        self.payment_filter.bind(text=lambda instance, value: self.load_products())
        filter_layout.add_widget(self.payment_filter)
        
        self.expiry_filter_btn = ToggleButton(
            text='Expiring Soon',
            group='filters',
            allow_no_selection=True
        )
        self.expiry_filter_btn.bind(state=lambda instance, value: self.load_products())
        filter_layout.add_widget(self.expiry_filter_btn)
        
        self.low_stock_btn = ToggleButton(
            text='Low Stock',
            group='filters',
            allow_no_selection=True
        )
        self.low_stock_btn.bind(state=lambda instance, value: self.load_products())
        filter_layout.add_widget(self.low_stock_btn)
        self.layout.add_widget(filter_layout)

        # Search Box
        search_box = BoxLayout(size_hint_y=None, height=dp(40))
        self.search_input = TextInput(hint_text='Search products...')
        # Bind text input to trigger search on change
        self.search_input.bind(text=lambda instance, value: self.load_products())
        search_box.add_widget(self.search_input)
        search_box.add_widget(EnhancedButton(
            text='Clear', 
            size_hint_x=0.3,
            on_press=self.clear_search
        ))
        self.layout.add_widget(search_box)

        # Product List
        scroll_view = ScrollView()
        self.product_grid = GridLayout(cols=8, spacing=dp(5), size_hint_y=None)
        self.product_grid.bind(minimum_height=self.product_grid.setter('height'))
        scroll_view.add_widget(self.product_grid)
        self.layout.add_widget(scroll_view)

        self.layout.add_widget(EnhancedButton(
            text="Back",
            on_press=lambda x: setattr(self.manager, 'current', 'menu')
        ))

        self.add_widget(self.layout)
        self.load_products()

    def clear_search(self, instance):
        self.search_input.text = ''
        self.load_products()

    def on_enter(self, *args):
        self.load_products()

    def get_filtered_products(self):
        with connect_db() as conn:
            cursor = conn.cursor()
            query = "SELECT * FROM products WHERE 1=1"
            params = []

            # Payment mode filter
            payment_mode = self.payment_filter.text
            if payment_mode != 'All':
                query += " AND mode_of_payment = ?"
                params.append(payment_mode)
            
            # Expiring soon filter
            if self.expiry_filter_btn.state == 'down':
                query += " AND expiry_date BETWEEN date('now') AND date('now', '+30 days')"
            
            # Low stock filter
            if self.low_stock_btn.state == 'down':
                query += " AND quantity_remaining < 10"
            
            # Search term (case-insensitive)
            search_term = self.search_input.text.strip()
            if search_term:
                query += " AND (LOWER(name) LIKE ? OR LOWER(category) LIKE ?)"
                params.extend([f"%{search_term.lower()}%", f"%{search_term.lower()}%"])
            
            cursor.execute(query, params)
            products = cursor.fetchall()
            return products

    def load_products(self):
        self.product_grid.clear_widgets()
        headers = ["Drug Name", "Category", "Qty Remaining", "Qty to Expire",
                   "Price", "Expiry", "Payment", "Edit"]
        
        for header in headers:
            self.product_grid.add_widget(HeaderLabel(text=header))

        products = self.get_filtered_products()

        if not products:
            self.product_grid.add_widget(SecondaryLabel(
                text="No products found",
                col_span=8,
                halign='center'
            ))
            return

        for product in products:
            product_fields = [
                product[1], product[2], product[3], product[4],
                product[5], product[6], product[7]
            ]
            
            # Check stock and expiry status
            try:
                expiry_date = datetime.strptime(product_fields[5], '%Y-%m-%d').date()
                today = datetime.today().date()
                days_until_expiry = (expiry_date - today).days
                is_expiring = days_until_expiry <= 30
            except:
                is_expiring = False
            
            is_low_stock = product_fields[2] < 10
# Add product fields with styling
            for idx, value in enumerate(product_fields):
                text = str(value)
                label = SecondaryLabel(text=text)
                
                if idx == 2 and is_low_stock:  # Qty Remaining
                    label.color = get_color_from_hex('#FF0000')
                if idx == 5 and is_expiring:    # Expiry Date
                    label.color = get_color_from_hex('#FFA500')
                
                self.product_grid.add_widget(label)
            
            # Edit button
            edit_btn = EnhancedButton(
                text="Edit", 
                size_hint_x=None, 
                width=dp(80),
                on_press=lambda instance, pid=product[0]: self.open_edit_popup(pid)
            )
            self.product_grid.add_widget(edit_btn)

    def open_edit_popup(self, product_id):
        with connect_db() as conn:
            cursor = conn.cursor()
            cursor.execute("SELECT * FROM products WHERE id=?", (product_id,))
            product = cursor.fetchone()

        if not product:
            show_popup('Error', "Product not found")
            return

        layout = BoxLayout(orientation='vertical', spacing=10, padding=10)
        fields = {}

        labels = ["Drug Name", "Category", "Qty Remaining", "Qty to Expire",
                  "Price (MWK)", "Expiry Date", "Mode of Payment"]
        values = product[1:8]

        for label_text, value in zip(labels, values):
            box = BoxLayout(size_hint_y=None, height=dp(40))
            label = SecondaryLabel(text=label_text, size_hint_x=0.4)
            text_input = TextInput(text=str(value), multiline=False)
            fields[label_text] = text_input
            box.add_widget(label)
            box.add_widget(text_input)
            layout.add_widget(box)

        def update_product(instance):
            try:
                updated_values = (
                    fields["Drug Name"].text.strip(),
                    fields["Category"].text.strip(),
                    int(fields["Qty Remaining"].text.strip()),
                    int(fields["Qty to Expire"].text.strip()),
                    float(fields["Price (MWK)"].text.strip()),
                    fields["Expiry Date"].text.strip(),
                    fields["Mode of Payment"].text.strip(),
                    product_id
                )
                with connect_db() as conn:
                    cursor = conn.cursor()
                    cursor.execute('''
                        UPDATE products SET name=?, category=?, quantity_remaining=?, quantity_to_expire=?,
                        price=?, expiry_date=?, mode_of_payment=? WHERE id=?
                    ''', updated_values)
                    conn.commit()
                show_popup('Success', 'Product updated successfully')
                self.load_products()
                popup.dismiss()
            except Exception as e:
                show_popup('Error', str(e), size=(0.7, 0.3))

        save_btn = EnhancedButton(text="Save Changes")
        save_btn.bind(on_press=update_product)
        layout.add_widget(save_btn)

        popup = Popup(title='Edit Product', content=layout, size_hint=(0.9, 0.9))
        popup.open()

class RecordSaleScreen(Screen):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        layout = BoxLayout(orientation='vertical', padding=10, spacing=10)
        
        # Product selection
        self.product_spinner = Spinner(text='Select Product')
        layout.add_widget(self.product_spinner)
        
        # Quantity input
        qty_box_atomics = BoxLayout(size_hint_y=None, height=dp(40))
        qty_box.add_widget(SecondaryLabel(text='Quantity Sold:'))
        self.quantity_input = TextInput(multiline=False)
        qty_box.add_widget(self.quantity_input)
        layout.add_widget(qty_box)
        
        # Date input
        date_box = BoxLayout(size_hint_y=None, height=dp(40))
        date_box.add_widget(SecondaryLabel(text='Sale Date (YYYY-MM-DD):'))
        self.date_input = TextInput(text=datetime.now().strftime('%Y-%m-%d'), multiline=False)
        date_box.add_widget(self.date_input)
        layout.add_widget(date_box)
        
        # Buttons
        layout.add_widget(EnhancedButton(
            text='Record Sale',
            on_press=self.record_sale
        ))
        layout.add_widget(EnhancedButton(
            text='Back',
            on_press=lambda x: setattr(self.manager, 'current', 'menu')
        ))
        
        self.add_widget(layout)
        self.update_product_list()

    def on_enter(self, *args):
        self.update_product_list()

    def update_product_list(self):
        with connect_db() as conn:
            cursor = conn.cursor()
            cursor.execute("SELECT id, name FROM products")
            products = [f"{p[0]}: {p[1]}" for p in cursor.fetchall()]
        self.product_spinner.values = products
        if products:
            self.product_spinner.text = products[0]
        else:
            self.product_spinner.text = 'No products available'

    def record_sale(self, instance):
        if ':' not in self.product_spinner.text:
            show_popup('Error', 'Select a valid product')
            return
        
        try:
            product_id = int(self.product_spinner.text.split(':')[0])
            quantity = int(self.quantity_input.text)
            sale_date = self.date_input.text
            
            if not validate_date(sale_date):
                raise ValueError("Invalid date format")
            
            with connect_db() as conn:
                cursor = conn.cursor()
                
                # Check current stock
                cursor.execute("SELECT quantity_remaining FROM products WHERE id=?", (product_id,))
                current_stock = cursor.fetchone()[0]
                
                if quantity > current_stock:
                    raise ValueError("Insufficient stock")
                
                # Record sale
                cursor.execute('''
                    INSERT INTO sales (product_id, quantity_sold, sale_date)
                    VALUES (?, ?, ?)
                ''', (product_id, quantity, sale_date))
                
                # Update stock
                cursor.execute('''
                    UPDATE products SET quantity_remaining = ?
                    WHERE id = ?
                ''', (current_stock - quantity, product_id))
                conn.commit()
            
            show_popup('Success', 'Sale recorded')
            self.quantity_input.text = ''
            self.date_input.text = datetime.now().strftime('%Y-%m-%d')
            self.update_product_list()
            
        except Exception as e:
            show_popup('Error', str(e), size=(0.7, 0.3))

class ExpiryReminderScreen(Screen):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        layout = BoxLayout(orientation='vertical', padding=10, spacing=10)

        layout.add_widget(HeaderLabel(text='Drugs Expiring in 7 Days'))

        scroll = ScrollView()
        self.grid = GridLayout(cols=6, spacing=dp(5), size_hint_y=None)
        self.grid.bind(minimum_height=self.grid.setter('height'))
        scroll.add_widget(self.grid)
        layout.add_widget(scroll)

        layout.add_widget(EnhancedButton(
            text="Back",
            on_press=lambda x: setattr(self.manager, 'current', 'menu')
        ))

        self.add_widget(layout)

    def on_enter(self, *args):
        self.load_expiring_products()

    def load_expiring_products(self):
        self.grid.clear_widgets()
        headers = ["Drug Name", "Category", "Qty Remaining", "Expiry Date", "Price", "Payment"]
        for h in headers:
            self.grid.add_widget(HeaderLabel(text=h))

        with connect_db() as conn:
            cursor = conn.cursor()
            cursor.execute('''
                SELECT name, category, quantity_remaining, expiry_date, price, mode_of_payment
                FROM products 
                WHERE expiry_date BETWEEN date('now') AND date('now', '+7 days')
            ''')
            rows = cursor.fetchall()

        for row in rows:
            for value in row:
                self.grid.add_widget(SecondaryLabel(text=str(value)))

class LowStockAlertsScreen(Screen):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        layout = BoxLayout(orientation='vertical', padding=10, spacing=10)

        layout.add_widget(HeaderLabel(text='Low Stock Alerts (Qty < 10)'))

        scroll = ScrollView()
        self.grid = GridLayout(cols=6, spacing=dp(5), size_hint_y=None)
        self.grid.bind(minimum_height=self.grid.setter('height'))
        scroll.add_widget(self.grid)
        layout.add_widget(scroll)

        layout.add_widget(EnhancedButton(
            text="Back",
            on_press=lambda x: setattr(self.manager, 'current', 'menu')
        ))

        self.add_widget(layout)

    def on_enter(self, *args):
        self.load_low_stock_products()

    def load_low_stock_products(self):
        self.grid.clear_widgets()
        headers = ["Drug Name", "Category", "Qty Remaining", "Expiry Date", "Price", "Payment"]
        for h in headers:
            self.grid.add_widget(HeaderLabel(text=h))

        with connect_db() as conn:
            cursor = conn.cursor()
            cursor.execute('''
                SELECT name, category, quantity_remaining, expiry_date, price, mode_of_payment
                FROM products 
                WHERE quantity_remaining < 10
            ''')
            rows = cursor.fetchall()

        for row in rows:
            for value in row:
                label = SecondaryLabel(text=str(value))
                # Highlight low stock in red
                if value == row[2]:  # Qty Remaining column
                    try:
                        if int(value) < 10:
                            label.color = get_color_from_hex('#FF0000')
                    except ValueError:
                        pass
                self.grid.add_widget(label)
# ======================
# MAIN APPLICATION
# ======================
class PharmacyInventoryApp(App):
    def build(self):
        sm = ScreenManager()
        sm.add_widget(MenuScreen(name='menu'))
        sm.add_widget(AddProductScreen(name='add_product'))
        sm.add_widget(ViewProductsScreen(name='view_products'))
        sm.add_widget(RecordSaleScreen(name='record_sale'))
        sm.add_widget(ExpiryReminderScreen(name='expiry_reminder'))
        sm.add_widget(LowStockAlertsScreen(name='low_stock_alerts'))
        return sm

if __name__ == '__main__':
    PharmacyInventoryApp().run()