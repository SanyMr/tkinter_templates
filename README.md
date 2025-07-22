import tkinter as tk
from tkinter import ttk
from PIL import Image, ImageDraw, ImageTk


class DualTreeViewWidget:
    def __init__(self, parent, columns_names, attributs_names_list, model_objects):
        self.parent = parent
        self.columns_names = columns_names
        self.attributs_names_list = attributs_names_list
        self.model_objects = model_objects

        self.assignments = {}
        self.item_colors = {}
        self.next_block_number = 1
        self.next_address = 1

        self.selected_a_item = None
        self.selected_b_item = None

        self.setup_widget()
        self.populate_treeview_a()

    def setup_widget(self):
        self.main_frame = ttk.Frame(self.parent)
        self.main_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)

        self.left_frame = ttk.LabelFrame(self.main_frame, text="Objects")
        self.middle_frame = ttk.Frame(self.main_frame)
        self.right_frame = ttk.LabelFrame(self.main_frame, text="Blocks")

        self.left_frame.grid(row=0, column=0, sticky="nsew", padx=(0, 5))
        self.middle_frame.grid(row=0, column=1, sticky="ns", padx=10)
        self.right_frame.grid(row=0, column=2, sticky="nsew", padx=(5, 0))

        self.main_frame.columnconfigure(0, weight=2)
        self.main_frame.columnconfigure(2, weight=1)
        self.main_frame.rowconfigure(0, weight=1)

        self.setup_treeview_a()
        self.setup_arrows()
        self.setup_treeview_b()

    def setup_treeview_a(self):
        self.treeview_a = ttk.Treeview(self.left_frame, columns=self.columns_names, show='tree headings')
        self.treeview_a.column("#0", width=50)
        self.treeview_a.heading("#0", text="ID")

        for col in self.columns_names:
            self.treeview_a.column(col, width=100)
            self.treeview_a.heading(col, text=col)

        self.treeview_a.grid(row=0, column=0, sticky="nsew")
        vsb = ttk.Scrollbar(self.left_frame, orient=tk.VERTICAL, command=self.treeview_a.yview)
        hsb = ttk.Scrollbar(self.left_frame, orient=tk.HORIZONTAL, command=self.treeview_a.xview)
        self.treeview_a.configure(yscrollcommand=vsb.set, xscrollcommand=hsb.set)

        vsb.grid(row=0, column=1, sticky="ns")
        hsb.grid(row=1, column=0, sticky="ew")
        self.left_frame.rowconfigure(0, weight=1)
        self.left_frame.columnconfigure(0, weight=1)

        self.treeview_a.bind('<<TreeviewSelect>>', self.on_treeview_a_select)

    def setup_treeview_b(self):
        top_btns = ttk.Frame(self.right_frame)
        top_btns.grid(row=0, column=0, sticky="ew")

        ttk.Button(top_btns, text="Add Block", command=self.add_block).pack(side=tk.LEFT)
        ttk.Button(top_btns, text="Delete Block", command=self.delete_block).pack(side=tk.LEFT, padx=5)

        self.treeview_b = ttk.Treeview(self.right_frame, columns=("Block", "Address"), show='tree headings')
        self.treeview_b.column("Block", width=120)
        self.treeview_b.column("Address", width=80)
        self.treeview_b.heading("Block", text="Block")
        self.treeview_b.heading("Address", text="Address")
        self.treeview_b.tag_configure("block_tag", background="yellow")
        self.treeview_b.tag_configure("child_tag", background="white")

        self.treeview_b.grid(row=1, column=0, sticky="nsew")
        vsb = ttk.Scrollbar(self.right_frame, orient=tk.VERTICAL, command=self.treeview_b.yview)
        hsb = ttk.Scrollbar(self.right_frame, orient=tk.HORIZONTAL, command=self.treeview_b.xview)
        self.treeview_b.configure(yscrollcommand=vsb.set, xscrollcommand=hsb.set)

        vsb.grid(row=1, column=1, sticky="ns")
        hsb.grid(row=2, column=0, sticky="ew")
        self.right_frame.columnconfigure(0, weight=1)
        self.right_frame.rowconfigure(1, weight=1)

        self.treeview_b.bind('<<TreeviewSelect>>', self.on_treeview_b_select)

    def setup_arrows(self):
        self.right_arrow_img = self.create_arrow_image("right")
        self.left_arrow_img = self.create_arrow_image("left")

        tk.Button(self.middle_frame, image=self.right_arrow_img, command=self.move_right).pack(pady=(100, 10))
        tk.Button(self.middle_frame, image=self.left_arrow_img, command=self.move_left).pack(pady=10)

    def create_arrow_image(self, direction):
        img = Image.new('RGBA', (30, 20), (255, 255, 255, 0))
        draw = ImageDraw.Draw(img)
        if direction == "right":
            draw.polygon([(5, 5), (25, 10), (5, 15)], fill="black")
        else:
            draw.polygon([(25, 5), (5, 10), (25, 15)], fill="black")
        return ImageTk.PhotoImage(img)

    def populate_treeview_a(self):
        self.treeview_a.tag_configure("skyblue", background="lightblue")
        self.treeview_a.tag_configure("white", background="white")
        self.treeview_a.tag_configure("orange", background="orange")
        self.treeview_a.tag_configure("gray", background="gray")

        for i, obj in enumerate(self.model_objects):
            values = [self.get_attr(obj, attr) for attr in self.attributs_names_list]
            values = [str(v) if v is not None else "" for v in values]
            item_id = self.treeview_a.insert("", tk.END, text=str(i + 1), values=values, tags=["skyblue"])
            self.assignments[item_id] = {}
            self.item_colors[item_id] = "skyblue"

    def get_attr(self, obj, attr):
        try:
            for part in attr.split('.'):
                obj = getattr(obj, part)
            return obj
        except AttributeError:
            return None

    def add_block(self):
        block_name = f"Block_{self.next_block_number}"
        address = str(self.next_address)
        block_id = self.treeview_b.insert("", tk.END, values=(block_name, address), tags=["block_tag"])
        self.next_block_number += 1
        self.next_address += 1
        return block_id

    def delete_block(self):
        if self.selected_b_item and not self.treeview_b.parent(self.selected_b_item):
            for child in self.treeview_b.get_children(self.selected_b_item):
                vals = self.treeview_b.item(child)['values']
                if vals:
                    num = vals[0].split("_")[1]
                    for item in self.treeview_a.get_children():
                        if self.treeview_a.item(item)["text"] == num:
                            if self.selected_b_item in self.assignments[item]:
                                del self.assignments[item][self.selected_b_item]
                                self.update_item_color(item)
            self.treeview_b.delete(self.selected_b_item)
            self.selected_b_item = None

    def on_treeview_a_select(self, _): self.selected_a_item = self.treeview_a.selection()[0] if self.treeview_a.selection() else None
    def on_treeview_b_select(self, _): self.selected_b_item = self.treeview_b.selection()[0] if self.treeview_b.selection() else None

    def move_right(self):
        if not self.selected_a_item or not self.selected_b_item: return
        if self.treeview_b.parent(self.selected_b_item): return

        if self.selected_b_item not in self.assignments[self.selected_a_item]:
            self.assignments[self.selected_a_item][self.selected_b_item] = 0
        self.assignments[self.selected_a_item][self.selected_b_item] += 1

        text = self.treeview_a.item(self.selected_a_item)['text']
        values = self.treeview_a.item(self.selected_a_item)['values']
        child_values = [f"Item_{text}"] + list(values[:1])
        self.treeview_b.insert(self.selected_b_item, tk.END, values=child_values, tags=["child_tag"])
        self.update_item_color(self.selected_a_item)

    def move_left(self):
        if not self.selected_b_item: return
        parent = self.treeview_b.parent(self.selected_b_item)
        if not parent: return

        vals = self.treeview_b.item(self.selected_b_item)['values']
        if vals and vals[0].startswith("Item_"):
            item_num = vals[0].split("_")[1]
            for item in self.treeview_a.get_children():
                if self.treeview_a.item(item)['text'] == item_num:
                    if parent in self.assignments[item]:
                        self.assignments[item][parent] -= 1
                        if self.assignments[item][parent] <= 0:
                            del self.assignments[item][parent]
                        self.update_item_color(item)
                        self.treeview_b.delete(self.selected_b_item)
                        self.selected_b_item = None
                    break

    def update_item_color(self, item_id):
        total = sum(self.assignments[item_id].values())
        color_map = {0: "skyblue", 1: "white", 2: "orange"}
        tag = color_map.get(total, "gray")

        self.treeview_a.item(item_id, tags=[tag])
        self.item_colors[item_id] = tag


# Démo
class ExampleModel:
    def __init__(self, name, value, category):
        self.name = name
        self.value = value
        self.category = category
        self.details = type('Details', (), {'code': f"{name[:2].upper()}{value}"})()


def demo():
    root = tk.Tk()
    root.geometry("1000x600")
    root.title("Dual TreeView Widget Demo")

    model_objects = [
        ExampleModel("Product A", 100, "Electronics"),
        ExampleModel("Product B", 200, "Electronics"),
        ExampleModel("Service X", 150, "Services"),
        ExampleModel("Service Y", 75, "Services"),
        ExampleModel("Item Z", 300, "Misc")
    ]

    DualTreeViewWidget(
        parent=root,
        columns_names=["Name", "Value", "Category", "Code"],
        attributs_names_list=["name", "value", "category", "details.code"],
        model_objects=model_objects
    )

    root.mainloop()


if __name__ == "__main__":
    demo()
