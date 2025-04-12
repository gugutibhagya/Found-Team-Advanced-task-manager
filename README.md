a# Found-Team-Advanced-task-manager  import click
import sqlite3
from datetime import datetime
import json
from typing import Optional, List, Dict
import sys
import os
from enum import Enum

# Enums for task properties
class Priority(Enum):
    URGENT = 4
    HIGH = 3
    MEDIUM = 2
    LOW = 1

class Status(Enum):
    TODO = "Todo"
    IN_PROGRESS = "In Progress"
    COMPLETED = "Completed"
    BLOCKED = "Blocked"

# Database setup
def init_db():
    conn = sqlite3.connect('taskmanager.db')
    c = conn.cursor()
    
    # Tasks table
    c.execute('''CREATE TABLE IF NOT EXISTS tasks
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  title TEXT NOT NULL,
                  description TEXT,
                  due_date TEXT,
                  priority INTEGER,
                  status TEXT,
                  category TEXT,
                  created_at TEXT,
                  updated_at TEXT)''')
    
    # Comments table
    c.execute('''CREATE TABLE IF NOT EXISTS comments
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  task_id INTEGER,
                  author TEXT,
                  content TEXT,
                  created_at TEXT,
                  FOREIGN KEY(task_id) REFERENCES tasks(id))''')
    
    # Assignments table
    c.execute('''CREATE TABLE IF NOT EXISTS assignments
                 (task_id INTEGER,
                  assignee TEXT,
                  assigned_at TEXT,
                  PRIMARY KEY(task_id, assignee))''')
    
    conn.commit()
    conn.close()

# CLI Application
@click.group()
def cli():
    """Advanced Command-Line Task Manager"""
    init_db()

# Task CRUD Operations
@cli.command()
@click.argument('title')
@click.option('--description', '-d', help='Task description')
@click.option('--due', '-D', help='Due date (YYYY-MM-DD or YYYY-MM-DD HH:MM)')
@click.option('--priority', '-p', type=click.Choice(['urgent', 'high', 'medium', 'low']), 
              default='medium', help='Task priority')
@click.option('--category', '-c', help='Task category')
def add(title, description, due, priority, category):
    """Add a new task"""
    conn = sqlite3.connect('taskmanager.db')
    c = conn.cursor()
    
    try:
        due_date = datetime.strptime(due, '%Y-%m-%d %H:%M') if due else None
    except ValueError:
        try:
            due_date = datetime.strptime(due, '%Y-%m-%d') if due else None
        except ValueError:
            click.echo("Invalid date format. Use YYYY-MM-DD or YYYY-MM-DD HH:MM")
            return
    
    priority_value = Priority[priority.upper()].value
    created_at = datetime.now().isoformat()
    
    c.execute('''INSERT INTO tasks 
                 (title, description, due_date, priority, status, category, created_at, updated_at)
                 VALUES (?, ?, ?, ?, ?, ?, ?, ?)''',
              (title, description, due_date.isoformat() if due_date else None,
               priority_value, Status.TODO.value, category, created_at, created_at))
    
    task_id = c.lastrowid
    conn.commit()
    conn.close()
    
    click.echo(f"Task added with ID: {task_id}")

@cli.command()
@click.option('--status', '-s', type=click.Choice(['todo', 'in-progress', 'completed', 'blocked']),
              help='Filter by status')
@click.option('--priority', '-p', type=click.Choice(['urgent', 'high', 'medium', 'low']),
              help='Filter by priority')
@click.option('--category', '-c', help='Filter by category')
@click.option('--due', '-D', help='Filter by due date (before YYYY-MM-DD)')
def list(status, priority, category, due):
    """List all tasks"""
    conn = sqlite3.connect('taskmanager.db')
    c = conn.cursor()
    
    query = "SELECT * FROM tasks WHERE 1=1"
    params = []
    
    if status:
        query += " AND status = ?"
        params.append(Status[status.upper().replace('-', '_')].value)
    if priority:
        query += " AND priority = ?"
        params.append(Priority[priority.upper()].value)
    if category:
        query += " AND category = ?"
        params.append(category)
    if due:
        query += " AND due_date <= ?"
        params.append(due + " 23:59:59")
    
    c.execute(query, params)
    tasks = c.fetchall()
    
    if not tasks:
        click.echo("No tasks found.")
        return
    
    # Display tasks in a formatted table
    click.echo("\n{:<5} {:<30} {:<10} {:<12} {:<15} {:<10}".format(
        "ID", "Title", "Priority", "Status", "Due Date", "Category"))
    click.echo("-" * 85)
    
    for task in tasks:
        due_date = datetime.fromisoformat(task[3]).strftime('%Y-%m-%d') if task[3] else "None"
        priority_name = Priority(task[4]).name.capitalize()
        click.echo("{:<5} {:<30} {:<10} {:<12} {:<15} {:<10}".format(
            task[0], task[1][:28] + "..." if len(task[1]) > 28 else task[1],
            priority_name, task[5], due_date, task[6] or ""))
    
    conn.close()

# Advanced Features
@cli.command()
@click.argument('task_id', type=int)
@click.argument('assignee')
def assign(task_id, assignee):
    """Assign task to a team member"""
    conn = sqlite3.connect('taskmanager.db')
    c = conn.cursor()
    
    # Check if task exists
    c.execute("SELECT 1 FROM tasks WHERE id = ?", (task_id,))
    if not c.fetchone():
        click.echo(f"Task with ID {task_id} not found.")
        return
    
    assigned_at = datetime.now().isoformat()
    
    try:
        c.execute('''INSERT OR REPLACE INTO assignments
                     (task_id, assignee, assigned_at)
                     VALUES (?, ?, ?)''',
                  (task_id, assignee, assigned_at))
        conn.commit()
        click.echo(f"Task {task_id} assigned to {assignee}")
    except sqlite3.Error as e:
        click.echo(f"Error assigning task: {e}")
    finally:
        conn.close()

@cli.command()
@click.argument('task_id', type=int)
@click.argument('content')
@click.option('--author', '-a', default="System", help='Comment author')
def comment(task_id, content, author):
    """Add comment to a task"""
    conn = sqlite3.connect('taskmanager.db')
    c = conn.cursor()
    
    # Check if task exists
    c.execute("SELECT 1 FROM tasks WHERE id = ?", (task_id,))
    if not c.fetchone():
        click.echo(f"Task with ID {task_id} not found.")
        return
    
    created_at = datetime.now().isoformat()
    
    try:
        c.execute('''INSERT INTO comments
                     (task_id, author, content, created_at)
                     VALUES (?, ?, ?, ?)''',
                  (task_id, author, content, created_at))
        conn.commit()
        click.echo(f"Comment added to task {task_id}")
    except sqlite3.Error as e:
        click.echo(f"Error adding comment: {e}")
    finally:
        conn.close()

# Reporting
@cli.command()
@click.option('--format', '-f', type=click.Choice(['text', 'json', 'csv']), 
              default='text', help='Output format')
def report(format):
    """Generate task statistics report"""
    conn = sqlite3.connect('taskmanager.db')
    c = conn.cursor()
    
    # Get counts by status
    c.execute('''SELECT status, COUNT(*) FROM tasks GROUP BY status''')
    status_counts = dict(c.fetchall())
    
    # Get counts by priority
    c.execute('''SELECT priority, COUNT(*) FROM tasks GROUP BY priority''')
    priority_counts = {Priority(p).name.capitalize(): count for p, count in c.fetchall()}
    
    # Get overdue tasks count
    c.execute('''SELECT COUNT(*) FROM tasks 
                 WHERE due_date < ? AND status != ?''',
              (datetime.now().isoformat(), Status.COMPLETED.value))
    overdue_count = c.fetchone()[0]
    
    conn.close()
    
    # Generate report based on format
    report_data = {
        'status_distribution': status_counts,
        'priority_distribution': priority_counts,
        'overdue_tasks': overdue_count,
        'total_tasks': sum(status_counts.values())
    }
    
    if format == 'json':
        click.echo(json.dumps(report_data, indent=2))
    elif format == 'csv':
        click.echo("Metric,Count")
        for metric, count in report_data.items():
            if isinstance(count, dict):
                for submetric, subcount in count.items():
                    click.echo(f"{metric}.{submetric},{subcount}")
            else:
                click.echo(f"{metric},{count}")
    else:
        click.echo("\n=== Task Manager Report ===")
        click.echo(f"\nTotal Tasks: {report_data['total_tasks']}")
        click.echo(f"Overdue Tasks: {report_data['overdue_tasks']}")
        
        click.echo("\nStatus Distribution:")
        for status, count in report_data['status_distribution'].items():
            click.echo(f"  {status}: {count}")
        
        click.echo("\nPriority Distribution:")
        for priority, count in report_data['priority_distribution'].items():
            click.echo(f"  {priority}: {count}")

if __name__ == '__main__':
    cli()
