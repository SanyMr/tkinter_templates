import tkinter as tk
from tkinter import ttk


class DynamicRowWidget:
    def __init__(self, root, initial_data=None, rows_per_page=10):
        self.root = root
        self.root.title("Dynamic Row Widget")
        self.root.geometry("700x500")

        # Data management
        self.all_data = []  # Store all data
        self.displayed_rows = []  # Currently displayed row widgets
        self.row_counter = 0

        # Pagination settings
        self.rows_per_page = rows_per_page
        self.current_page = 1
        self.total_pages = 1

        # Create main frame
        self.main_frame = ttk.Frame(root)
        self.main_frame.pack(fill=tk.BOTH, expand=True)

        # Create toolbar
        self.create_toolbar()

        # Create scrollable frame for rows
        self.create_scrollable_frame()

        # Create pagination controls
        self.create_pagination_controls()

        # Load initial data if provided
        if initial_data:
            self.load_data(initial_data)
        else:
            self.update_pagination_display()

    def create_toolbar(self):
        """Create toolbar with Add button"""
        toolbar = ttk.Frame(self.main_frame)
        toolbar.pack(fill=tk.X, pady=(5, 10), padx=10)

        # Add button
        self.add_btn = ttk.Button(toolbar, text="Add Row", command=self.add_row)
        self.add_btn.pack(side=tk.LEFT, padx=(0, 5))

        # Remove selected button
        self.remove_btn = ttk.Button(toolbar, text="Remove Selected", command=self.remove_selected_rows)
        self.remove_btn.pack(side=tk.LEFT, padx=(0, 5))

        # Clear all button
        self.clear_btn = ttk.Button(toolbar, text="Clear All", command=self.clear_all_rows)
        self.clear_btn.pack(side=tk.LEFT)

    def create_scrollable_frame(self):
        """Create scrollable frame for dynamic rows"""
        # Create canvas and scrollbar
        self.canvas = tk.Canvas(self.main_frame, bg='white')
        self.scrollbar = ttk.Scrollbar(self.main_frame, orient="vertical", command=self.canvas.yview)

        # Create frame inside canvas
        self.scrollable_frame = ttk.Frame(self.canvas)

        # Configure scrolling
        self.scrollable_frame.bind(
            "<Configure>",
            lambda e: self.canvas.configure(scrollregion=self.canvas.bbox("all"))
        )

        self.canvas_frame = self.canvas.create_window((0, 0), window=self.scrollable_frame, anchor="nw")
        self.canvas.configure(yscrollcommand=self.scrollbar.set)

        # Bind canvas resize to expand frame width
        self.canvas.bind("<Configure>", self._on_canvas_configure)

        # Pack canvas and scrollbar
        self.canvas.pack(side="left", fill="both", expand=True)
        self.scrollbar.pack(side="right", fill="y")

        # Bind mousewheel to canvas
        self.canvas.bind("<MouseWheel>", self._on_mousewheel)

        # Create header
        self.create_header()

    def create_header(self):
        """Create header row"""
        header_frame = ttk.Frame(self.scrollable_frame)
        header_frame.pack(fill=tk.X, pady=(0, 5))

        # Header labels with grid layout for better space utilization
        header_frame.grid_columnconfigure(1, weight=1)  # Name column expands
        header_frame.grid_columnconfigure(2, weight=1)  # Category column expands

        ttk.Label(header_frame, text="Select").grid(row=0, column=0, padx=(5, 10), sticky="w")
        ttk.Label(header_frame, text="Name").grid(row=0, column=1, padx=(0, 10), sticky="w")
        ttk.Label(header_frame, text="Category").grid(row=0, column=2, padx=(0, 10), sticky="w")

        # Add separator
        separator = ttk.Separator(self.scrollable_frame, orient='horizontal')
        separator.pack(fill=tk.X, pady=(0, 5))

    def create_pagination_controls(self):
        """Create pagination control bar"""
        pagination_frame = ttk.Frame(self.main_frame)
        pagination_frame.pack(fill=tk.X, pady=5, padx=10)

        # Left side - Navigation buttons
        nav_frame = ttk.Frame(pagination_frame)
        nav_frame.pack(side=tk.LEFT)

        # Previous x10 button - FIXED: removed the parentheses that were causing recursion
        self.prev_x10_btn = ttk.Button(nav_frame, text="<<", width=4, command=self.prev_x10_pages)
        self.prev_x10_btn.pack(side=tk.LEFT, padx=(0, 2))

        # Previous button
        self.prev_btn = ttk.Button(nav_frame, text="<", width=4, command=self.prev_page)
        self.prev_btn.pack(side=tk.LEFT, padx=(0, 2))

        # Next button
        self.next_btn = ttk.Button(nav_frame, text=">", width=4, command=self.next_page)
        self.next_btn.pack(side=tk.LEFT, padx=(0, 2))

        # Next x10 button
        self.next_x10_btn = ttk.Button(nav_frame, text=">>", width=4, command=self.next_x10_pages)
        self.next_x10_btn.pack(side=tk.LEFT)

        # Center - Page selection
        page_frame = ttk.Frame(pagination_frame)
        page_frame.pack(side=tk.LEFT, padx=20)

        ttk.Label(page_frame, text="Page:").pack(side=tk.LEFT, padx=(0, 5))

        # Current page combobox
        self.page_var = tk.StringVar()
        self.page_combo = ttk.Combobox(page_frame, textvariable=self.page_var, width=5, state="readonly")
        self.page_combo.pack(side=tk.LEFT, padx=(0, 5))
        self.page_combo.bind('<<ComboboxSelected>>', self.on_page_select)

        # Total pages label
        self.total_pages_label = ttk.Label(page_frame, text="of 1")
        self.total_pages_label.pack(side=tk.LEFT, padx=(5, 0))

        # Right side - Rows per page
        rows_frame = ttk.Frame(pagination_frame)
        rows_frame.pack(side=tk.RIGHT)

        ttk.Label(rows_frame, text="Rows per page:").pack(side=tk.LEFT, padx=(0, 5))

        self.rows_per_page_var = tk.StringVar(value=str(self.rows_per_page))
        self.rows_per_page_combo = ttk.Combobox(
            rows_frame,
            textvariable=self.rows_per_page_var,
            values=["5", "10", "20", "50", "100"],
            width=5,
            state="readonly"
        )
        self.rows_per_page_combo.pack(side=tk.LEFT)
        self.rows_per_page_combo.bind('<<ComboboxSelected>>', self.on_rows_per_page_change)

    def add_row(self, name=None, category=None, selected=False):
        """Add a new row to the data (not directly to display)"""
        self.row_counter += 1

        # Use provided values or defaults
        if name is None:
            name = f"Item {self.row_counter}"
        if category is None:
            category = "Category A"

        # Add to data storage
        row_data = {
            'name': name,
            'category': category,
            'selected': selected
        }
        self.all_data.append(row_data)

        # Calculate which page the new row will be on
        new_row_index = len(self.all_data) - 1
        target_page = (new_row_index // self.rows_per_page) + 1

        # Navigate to the page containing the new row
        self.current_page = target_page

        # Recalculate pagination and refresh display
        self.calculate_pagination()
        self.display_current_page()
        self.update_pagination_controls()

    def create_display_row(self, data, index):
        """Create a display row widget from data"""
        row_frame = ttk.Frame(self.scrollable_frame)
        row_frame.pack(fill=tk.X, pady=2, padx=5)

        # Configure grid columns to expand
        row_frame.grid_columnconfigure(1, weight=1)  # Name column expands
        row_frame.grid_columnconfigure(2, weight=1)  # Category column expands

        # Create row components
        row_widgets = {}

        # Checkbutton
        row_widgets['selected'] = tk.BooleanVar(value=data['selected'])
        checkbutton = ttk.Checkbutton(row_frame, variable=row_widgets['selected'])
        checkbutton.grid(row=0, column=0, padx=(0, 10), sticky="w")

        # Name entry
        row_widgets['name'] = tk.StringVar(value=data['name'])
        name_entry = ttk.Entry(row_frame, textvariable=row_widgets['name'])
        name_entry.grid(row=0, column=1, padx=(0, 10), sticky="ew")

        # Combobox
        row_widgets['category'] = tk.StringVar(value=data['category'])
        category_combo = ttk.Combobox(
            row_frame,
            textvariable=row_widgets['category'],
            values=["Category A", "Category B", "Category C", "Category D"],
            state="readonly"
        )
        category_combo.grid(row=0, column=2, padx=(0, 10), sticky="ew")

        # Store references
        row_widgets['frame'] = row_frame
        row_widgets['data_index'] = index  # Index in all_data

        return row_widgets

    def remove_selected_rows(self):
        """Remove rows that are checked"""
        # Sync displayed data back to all_data first
        self.sync_displayed_to_data()

        # Remove selected items from all_data
        self.all_data = [item for item in self.all_data if not item['selected']]

        # Update display
        self.update_pagination_display()

    def clear_all_rows(self):
        """Remove all rows"""
        self.all_data.clear()
        for row_widget in self.displayed_rows:
            row_widget['frame'].destroy()
        self.displayed_rows.clear()
        self.row_counter = 0
        self.current_page = 1
        self.update_pagination_display()

    def sync_displayed_to_data(self):
        """Sync changes from displayed rows back to all_data"""
        for row_widget in self.displayed_rows:
            data_index = row_widget['data_index']
            if data_index < len(self.all_data):
                self.all_data[data_index]['name'] = row_widget['name'].get()
                self.all_data[data_index]['category'] = row_widget['category'].get()
                self.all_data[data_index]['selected'] = row_widget['selected'].get()

    def load_data(self, data):
        """Load data from a list of dictionaries"""
        # Clear existing rows first
        self.clear_all_rows()

        for item in data:
            name = item.get('name', f'Item {self.row_counter + 1}')
            category = item.get('category', 'Category A')
            selected = item.get('selected', False)

            self.add_row(name=name, category=category, selected=selected)

    def get_all_data(self):
        """Get data from all rows - FIXED: now syncs and returns all_data"""
        # First sync any changes from displayed rows
        self.sync_displayed_to_data()
        # Return a copy of all data
        return [row_data.copy() for row_data in self.all_data]

    def _on_canvas_configure(self, event):
        """Handle canvas resize to expand frame width"""
        canvas_width = event.width
        self.canvas.itemconfig(self.canvas_frame, width=canvas_width)

    def _on_mousewheel(self, event):
        """Handle mouse wheel scrolling"""
        self.canvas.yview_scroll(int(-1 * (event.delta / 120)), "units")

    def display_current_page(self):
        """Display rows for the current page"""
        # Clear existing displayed rows
        for row_widget in self.displayed_rows:
            row_widget['frame'].destroy()
        self.displayed_rows.clear()

        if not self.all_data:
            return

        # Calculate start and end indices
        start_idx = (self.current_page - 1) * self.rows_per_page
        end_idx = min(start_idx + self.rows_per_page, len(self.all_data))

        # Create display rows for current page
        for i in range(start_idx, end_idx):
            row_widget = self.create_display_row(self.all_data[i], i)
            self.displayed_rows.append(row_widget)

        # Update scroll region
        self.canvas.update_idletasks()
        self.canvas.configure(scrollregion=self.canvas.bbox("all"))

    def calculate_pagination(self):
        """Calculate total pages based on data and rows per page"""
        if not self.all_data:
            self.total_pages = 1
        else:
            self.total_pages = (len(self.all_data) + self.rows_per_page - 1) // self.rows_per_page

        # Ensure current page is valid
        if self.current_page > self.total_pages:
            self.current_page = max(1, self.total_pages)

    def update_pagination_controls(self):
        """Update pagination control states and values"""
        # Update page combobox values
        page_values = [str(i) for i in range(1, self.total_pages + 1)]
        self.page_combo['values'] = page_values
        self.page_var.set(str(self.current_page))

        # Update total pages label
        self.total_pages_label.config(text=f"of {self.total_pages}")

        # Update button states
        self.prev_btn.config(state="normal" if self.current_page > 1 else "disabled")
        self.prev_x10_btn.config(state="normal" if self.current_page > 10 else "disabled")
        self.next_btn.config(state="normal" if self.current_page < self.total_pages else "disabled")
        self.next_x10_btn.config(state="normal" if self.current_page <= self.total_pages - 10 else "disabled")

    def update_pagination_display(self):
        """Update the entire pagination display"""
        self.calculate_pagination()
        self.display_current_page()
        self.update_pagination_controls()

    # Pagination navigation methods
    def prev_page(self):
        if self.current_page > 1:
            # Sync current page data before changing
            self.sync_displayed_to_data()
            self.current_page -= 1
            self.update_pagination_display()

    def next_page(self):
        if self.current_page < self.total_pages:
            # Sync current page data before changing
            self.sync_displayed_to_data()
            self.current_page += 1
            self.update_pagination_display()

    def prev_x10_pages(self):
        # Sync current page data before changing
        self.sync_displayed_to_data()
        self.current_page = max(1, self.current_page - 10)
        self.update_pagination_display()

    def next_x10_pages(self):
        # Sync current page data before changing
        self.sync_displayed_to_data()
        self.current_page = min(self.total_pages, self.current_page + 10)
        self.update_pagination_display()

    def on_page_select(self, event=None):
        try:
            new_page = int(self.page_var.get())
            if 1 <= new_page <= self.total_pages:
                # Sync current page data before changing
                self.sync_displayed_to_data()
                self.current_page = new_page
                self.update_pagination_display()
        except ValueError:
            pass

    def on_rows_per_page_change(self, event=None):
        try:
            new_rows_per_page = int(self.rows_per_page_var.get())
            # Sync current page data before changing
            self.sync_displayed_to_data()
            self.rows_per_page = new_rows_per_page
            self.current_page = 1  # Reset to first page
            self.update_pagination_display()
        except ValueError:
            pass


# Example usage
if __name__ == "__main__":
    root = tk.Tk()

    # Example initial data
    sample_data = [
        {'name': 'Task 1', 'category': 'Category B', 'selected': True},
        {'name': 'Task 2', 'category': 'Category A', 'selected': False},
        {'name': 'Task 3', 'category': 'Category C', 'selected': False},
    ]

    # Create widget with initial data
    widget = DynamicRowWidget(root, initial_data=sample_data)

    # Add some example functionality to demonstrate data retrieval
    def print_data():
        data = widget.get_all_data()
        print("Current data:")
        for i, row in enumerate(data):
            print(f"Row {i + 1}: {row}")

    def load_new_data():
        new_data = [
            {'name': 'New Item 1', 'category': 'Category D', 'selected': False},
            {'name': 'New Item 2', 'category': 'Category B', 'selected': True},
        ]
        widget.load_data(new_data)

    # Add buttons to demonstrate functionality
    demo_frame = ttk.Frame(root)
    demo_frame.pack(fill=tk.X, padx=10, pady=5)

    ttk.Button(demo_frame, text="Print All Data", command=print_data).pack(side=tk.LEFT, padx=(0, 5))
    ttk.Button(demo_frame, text="Load New Data", command=load_new_data).pack(side=tk.LEFT)

    root.mainloop()
