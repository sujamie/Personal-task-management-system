# 任務管理系統  
基於 Python、MSSQL 與 Tkinter 圖形介面的任務管理系統。系統允許使用者新增、刪除、標記完成或未完成任務，並提供即時同步資料庫的功能。
<hr><br>
<h1>系統架構</h1> 

>![](https://github.com/sujamie/Personal-task-management-system/blob/main/%E4%BB%BB%E5%8B%99%E7%AE%A1%E7%90%86%E7%B3%BB%E7%B5%B1%E4%BB%8B%E9%9D%A2.png)
>![](https://github.com/sujamie/Personal-task-management-system/blob/main/%E6%96%B0%E5%A2%9E%E4%BB%BB%E5%8B%99%E4%BB%8B%E9%9D%A2.png)
>![](https://github.com/sujamie/Personal-task-management-system/blob/main/%E6%A8%99%E8%A8%98%E5%AE%8C%E6%88%90%E7%95%AB%E9%9D%A2.png)
本系統包含以下主要部分：  

資料庫連線模組：使用 pyodbc 連接 MSSQL 資料庫。  

裝飾器：計算函式執行時間。  

Task 類別：定義任務的屬性與狀態管理。  

TaskManager 類別：負責任務的資料庫操作。  

TaskApp 類別：基於 Tkinter 設計的圖形介面。  

主要功能說明  
引入模組
``` python
import pyodbc
import time
import tkinter as tk
from tkinter import messagebox
from datetime import datetime
```

1.資料庫連線  
``` python
# MSSQL 資料庫設定
DB_CONFIG = {
    "server": "伺服器名稱",
    "database": "資料庫名稱",
    "username": "使用者名稱",
    "password": "使用者密碼",
}

# 連線到資料庫
def get_db_connection():
    conn = pyodbc.connect(
        f"DRIVER={{SQL Server}};SERVER={DB_CONFIG['server']};DATABASE={DB_CONFIG['database']};UID={DB_CONFIG['username']};PWD={DB_CONFIG['password']}"
    )
    return conn
```

2.使用裝飾器測試函式執行時間
``` python
def timer_decorator(func):
    def wrapper(*arg, **kwargs):
        start_time = time.time()
        result = func(*arg, **kwargs)
        end_time = time.time()
        print(f"{func.__name__} 執行時間: {end_time - start_time:.4f} 秒 ")
        return result
    return wrapper
```

3.定義任務類別
``` python
class Task:
    def __init__(self, title, description, due_date, completed=False, priority=None):
        self.title = title
        self.description = description
        self.due_date = due_date
        self.completed = completed
        self.priority = priority
    
    def mark_completed(self):
        self.completed = True
    
    def mark_incomplete(self):
        self.completed = False  # 標記為未完成
    
    def __str__(self):
        status = "已完成" if self.completed else "未完成"
        priority_str = f" | 優先級: {self.priority}" if self.priority else ""
        return f"{self.title} | 描述: {self.description} | 截止日: {self.due_date} | 狀態: {status}{priority_str}"

```

4.任務管理系統
``` python
class TaskManager:
    def __init__(self):
        self.tasks = []
        self.load_tasks_from_db()

    def load_tasks_from_db(self):
        """從 MSSQL 載入所有任務"""
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT title, description, due_date, completed, priority FROM tasks")
        self.tasks = [Task(title, desc, due, bool(comp), priority) for title, desc, due, comp, priority in cursor.fetchall()]
        conn.close()



    @timer_decorator
    def add_task(self, task):
        
        """新增任務到 MSSQL"""
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute(
            "INSERT INTO tasks (title, description, due_date, completed, priority) VALUES (?, ?, ?, ?, ?)",
            (task.title, task.description,  datetime.strptime(task.due_date, "%Y-%m-%d"), task.completed, task.priority),
        )
        conn.commit()
        conn.close()
        self.tasks.append(task)
        print(f"任務「{task.title}」已新增!")

    @timer_decorator
    def list_tasks(self):
        """回傳排序後的任務列表"""
        return sorted(self.tasks, key=lambda t: t.priority if t.priority is not None else 4)

    @timer_decorator
    def mark_task_completed(self, title):
        """將 MSSQL 中的任務標記為已完成"""
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("UPDATE tasks SET completed = 1 WHERE title = ?", (title,))
        conn.commit()
        conn.close()

        for task in self.tasks:
            if task.title == title:
                task.mark_completed()
                print(f"任務「{title}」已完成!")
                return
        print(f"無法找到名為「{title}」的任務!")

    @timer_decorator
    def mark_task_incomplete(self, title):
        """將 MSSQL 中的任務標記為未完成"""
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("UPDATE tasks SET completed = 0 WHERE title = ?", (title,))
        conn.commit()
        conn.close()
        for task in self.tasks:
            if task.title == title:
                task.mark_incomplete()
                print(f"任務「{title}」標記為未完成!")
                return
        print(f"無法找到名為「{title}」的任務!")

    @timer_decorator
    def delete_task(self, title):
        """從 MSSQL 刪除任務"""
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("DELETE FROM tasks WHERE title = ?", (title,))
        conn.commit()
        conn.close()

        self.tasks = [task for task in self.tasks if task.title != title]
        print(f"任務「{title}」已刪除!")
```

5.圖形介面
``` python
class TaskApp:
    def __init__(self, root):
        self.manager = TaskManager()
        self.root = root
        self.root.title("任務管理系統")
        

        self.listbox = tk.Listbox(root, width=root.winfo_screenwidth(), height=15) # 加寬以顯示完整資訊
        self.listbox.pack()

        button_width = 15  # 設定一致的按鈕寬度
        tk.Button(root, text="新增任務", width=button_width, font=('Arial',20,'bold', 'italic'), command=self.add_task).pack()
        tk.Button(root, text="標記完成", width=button_width, font=('Arial',20,'bold', 'italic'), command=self.mark_completed).pack()
        tk.Button(root, text="標記未完成", width=button_width, font=('Arial',20,'bold', 'italic'), command=self.mark_incomplete).pack()
        tk.Button(root, text="刪除任務", width=button_width, font=('Arial',20,'bold', 'italic'), command=self.delete_task).pack()

        self.update_listbox()

    def update_listbox(self):
        """更新任務列表"""
        self.listbox.delete(0, tk.END)
        for idx, task in enumerate(self.manager.list_tasks()):
            self.listbox.insert(tk.END, str(task))
            color = "green" if task.completed else "red"
            self.listbox.itemconfig(self.listbox.size() - 1, {'fg': color})  # 設定顏色
            self.listbox.insert(tk.END, "-" * root.winfo_screenwidth())

    def add_task(self):
        def submit():
            title = title_entry.get()
            description = description_entry.get()
            due_date = due_date_entry.get()
            try:
                priority = int(priority_entry.get()) if priority_entry.get() else None
                if priority and priority not in [1, 2, 3]:
                    raise ValueError("優先級必須是 1-3 之間的數字")
            except ValueError as e:
                messagebox.showerror("錯誤", str(e))
                return

            if not title:
                messagebox.showerror("錯誤", "任務名稱不能為空")
                return
            
            try:
                datetime.strptime(due_date, "%Y-%m-%d")
            except ValueError:
                messagebox.showerror("錯誤", "日期格式錯誤，請使用 YYYY-MM-DD 格式")
                return

            task = Task(title, description, due_date, False, priority)
            self.manager.add_task(task)
            self.update_listbox()
            input_window.destroy()


        input_window = tk.Toplevel(self.root)
        input_window.title("新增任務")
        input_window.geometry("400x300")
        input_window.resizable(True, True)

        tk.Label(input_window, text="任務名稱:").pack()
        title_entry = tk.Entry(input_window,width=root.winfo_screenwidth())
        title_entry.pack()

        tk.Label(input_window, text="任務描述:").pack()
        description_entry = tk.Entry(input_window,width=root.winfo_screenwidth())
        description_entry.pack()

        tk.Label(input_window, text="截止日期 (YYYY-MM-DD):").pack()
        due_date_entry = tk.Entry(input_window,width=root.winfo_screenwidth())
        due_date_entry.pack()

        tk.Label(input_window, text="優先級 (1-3, 可選):").pack()
        priority_entry = tk.Entry(input_window,width=root.winfo_screenwidth())
        priority_entry.pack()

        submit_button = tk.Button(input_window, text="提交",width=root.winfo_screenwidth(), command=submit)
        submit_button.pack()
    def mark_completed(self):
        """標記選定的任務為完成"""
        selected = self.listbox.curselection()
        if not selected:
            messagebox.showwarning("警告", "請選擇要標記的任務!")
            return
        task_info = self.listbox.get(selected[0])
        task_title = task_info.split("|")[0].strip()
        self.manager.mark_task_completed(task_title)
        self.update_listbox()

    def mark_incomplete(self):
        """標記選定的任務為未完成"""
        selected = self.listbox.curselection()
        if not selected:
            messagebox.showwarning("警告", "請選擇要標記的任務!")
            return
        task_info = self.listbox.get(selected[0])
        task_title = task_info.split("|")[0].strip()
        self.manager.mark_task_incomplete(task_title)
        self.update_listbox()

    def delete_task(self):
        """刪除選定的任務"""
        selected = self.listbox.curselection()
        if not selected:
            messagebox.showwarning("警告", "請選擇要刪除的任務!")
            return
        task_info = self.listbox.get(selected[0])
        task_title = task_info.split("|")[0].strip()
        self.manager.delete_task(task_title)
        self.update_listbox()
```
6.主程式執行
```python
if __name__ == "__main__":
    root = tk.Tk()
    app = TaskApp(root)
    root.mainloop()
```
