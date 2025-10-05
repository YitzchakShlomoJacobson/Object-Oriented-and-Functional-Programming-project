# Object-Oriented-and-Functional-Programming-project

"""habit.py
Habit class representing a single habit and its completion events.
"""

from datetime import datetime, timedelta
from typing import List, Optional, Tuple


class Habit:
    """
    Represents a habit with events (completion timestamps).

    Attributes
    ----------
    name : str
        the name of the habit
    periodicity : str
        'daily' or 'weekly'
    created_at : datetime
        timestamp when habit was created
    completions : List[datetime]
        list of completion timestamps
    """

    def __init__(self, name: str, periodicity: str):
        if periodicity not in ("daily", "weekly"):
            raise ValueError("periodicity must be 'daily' or 'weekly'")
        self.name = name
        self.periodicity = periodicity
        self.created_at = datetime.now()
        self.completions: List[datetime] = []

    def complete(self, when: Optional[datetime] = None):
        """Record a completion event (timestamp)."""
        when = when or datetime.now()
        self.completions.append(when)

    def _sorted_dates_desc(self) -> List[datetime]:
        return sorted(self.completions, reverse=True)

    @staticmethod
    def _iso_year_week(dt_date) -> Tuple[int, int]:
        y, w, _ = dt_date.isocalendar()
        return (y, w)

    @staticmethod
    def _prev_year_week(yw: Tuple[int, int]) -> Tuple[int, int]:
        year, week = yw
        # Handle week 1 -> previous year's last ISO week
        if week > 1:
            return (year, week - 1)
        else:
            # last ISO week of previous year can be 52 or 53
            prev_year = year - 1
            # find the ISO week number of Dec 28th (always in last ISO week of the year)
            from datetime import date
            last_week = date(prev_year, 12, 28).isocalendar()[1]
            return (prev_year, last_week)

    def get_streak(self, as_of: Optional[datetime] = None) -> int:
        """
        Compute current streak up to `as_of` (defaults to now).
        Streak defined as number of consecutive periods (days or ISO weeks)
        containing at least one completion event.
        """
        if not self.completions:
            return 0
        as_of = as_of or datetime.now()
        sorted_comps = self._sorted_dates_desc()

        if self.periodicity == "daily":
            seen_days = set(c.date() for c in sorted_comps)
            streak = 0
            check_date = as_of.date()
            # if there's no completion on as_of.date(), streak is zero
            while check_date in seen_days:
                streak += 1
                check_date = check_date - timedelta(days=1)
            return streak

        else:  # weekly
            # Build set of (year, week) for completions
            seen_weeks = set(self._iso_year_week(c.date()) for c in sorted_comps)
            streak = 0
            current = self._iso_year_week(as_of.date())
            while current in seen_weeks:
                streak += 1
                current = self._prev_year_week(current)
            return streak

    def is_broken(self, as_of: Optional[datetime] = None) -> bool:
        """Return True if current period has no completion."""
        return self.get_streak(as_of) == 0

    def to_dict(self):
        return {
            "name": self.name,
            "periodicity": self.periodicity,
            "created_at": self.created_at.isoformat(),
            "completions": [c.isoformat() for c in self.completions],
        }

    @classmethod
    def from_dict(cls, d):
        h = cls(d["name"], d["periodicity"])
        h.created_at = datetime.fromisoformat(d["created_at"])
        h.completions = [datetime.fromisoformat(s) for s in d.get("completions", [])]
        return h

    def __repr__(self):
        return f"Habit(name={self.name!r}, periodicity={self.periodicity!r}, completions={len(self.completions)})"

"""storage.py
Simple SQLite-based persistence for habits and completions.
"""

import sqlite3
from typing import List
from datetime import datetime
from habit import Habit
import os

# default DB file inside project folder
DB_FILE = os.path.join(os.path.dirname(__file__), "habits.db")


class Storage:
    def __init__(self, db_path: str = None):
        self.db_path = db_path or DB_FILE
        # enable foreign keys and text as is
        self.conn = sqlite3.connect(self.db_path, detect_types=sqlite3.PARSE_DECLTYPES)
        self.conn.execute("PRAGMA foreign_keys = ON;")
        self._ensure_tables()

    def _ensure_tables(self):
        cur = self.conn.cursor()
        cur.execute(
            """
            CREATE TABLE IF NOT EXISTS habits (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT UNIQUE NOT NULL,
                periodicity TEXT NOT NULL,
                created_at TEXT NOT NULL
            )
            """
        )
        cur.execute(
            """
            CREATE TABLE IF NOT EXISTS completions (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                habit_id INTEGER NOT NULL,
                completed_at TEXT NOT NULL,
                FOREIGN KEY(habit_id) REFERENCES habits(id) ON DELETE CASCADE
            )
            """
        )
        self.conn.commit()

    def save_habit(self, habit: Habit):
        """
        Insert or update habit and its completions.
        Simple approach: insert habit if missing, then delete its completions and re-insert.
        """
        cur = self.conn.cursor()
        cur.execute(
            "INSERT OR IGNORE INTO habits(name, periodicity, created_at) VALUES (?, ?, ?)",
            (habit.name, habit.periodicity, habit.created_at.isoformat()),
        )
        self.conn.commit()
        cur.execute("SELECT id FROM habits WHERE name = ?", (habit.name,))
        row = cur.fetchone()
        if row is None:
            raise RuntimeError("Failed to store habit")
        habit_id = row[0]
        # remove existing completions
        cur.execute("DELETE FROM completions WHERE habit_id = ?", (habit_id,))
        # insert completions
        for comp in habit.completions:
            cur.execute(
                "INSERT INTO completions(habit_id, completed_at) VALUES (?, ?)",
                (habit_id, comp.isoformat()),
            )
        self.conn.commit()

    def load_all_habits(self) -> List[Habit]:
        cur = self.conn.cursor()
        cur.execute("SELECT id, name, periodicity, created_at FROM habits ORDER BY name")
        rows = cur.fetchall()
        habits = []
        for r in rows:
            hid, name, periodicity, created_at = r
            cur.execute(
                "SELECT completed_at FROM completions WHERE habit_id = ? ORDER BY completed_at DESC", (hid,)
            )
            comps = [datetime.fromisoformat(row[0]) for row in cur.fetchall()]
            h = Habit(name, periodicity)
            h.created_at = datetime.fromisoformat(created_at)
            h.completions = comps
            habits.append(h)
        return habits

    def close(self):
        self.conn.close()

"""manager.py
HabitManager: in-memory collection with persistence integration.
"""

from typing import List, Optional
from habit import Habit
from storage import Storage


class HabitManager:
    def __init__(self, storage: Storage = None):
        self.storage = storage or Storage()
        self.habits: List[Habit] = self.storage.load_all_habits()

    def list_habits(self) -> List[Habit]:
        return self.habits

    def find(self, name: str) -> Optional[Habit]:
        for h in self.habits:
            if h.name == name:
                return h
        return None

    def create_habit(self, name: str, periodicity: str) -> Habit:
        if self.find(name):
            raise ValueError("Habit with this name already exists")
        h = Habit(name, periodicity)
        self.habits.append(h)
        self.storage.save_habit(h)
        return h

    def delete_habit(self, name: str) -> bool:
        h = self.find(name)
        if not h:
            return False
        self.habits.remove(h)
        # delete from DB
        cur = self.storage.conn.cursor()
        cur.execute("DELETE FROM habits WHERE name = ?", (name,))
        self.storage.conn.commit()
        return True

    def complete_habit(self, name: str):
        h = self.find(name)
        if not h:
            raise ValueError("Habit not found")
        h.complete()
        self.storage.save_habit(h)
        return h

"""analytics.py
Functional analytics for habits (pure functions).
"""

from typing import List, Tuple, Optional
from functools import reduce
from habit import Habit


def list_all_habits(habits: List[Habit]) -> List[str]:
    return list(map(lambda h: h.name, habits))


def list_by_periodicity(habits: List[Habit], periodicity: str) -> List[Habit]:
    return list(filter(lambda h: h.periodicity == periodicity, habits))


def longest_streak_for(habit: Habit) -> int:
    return habit.get_streak()


def longest_streak_all(habits: List[Habit]) -> Tuple[Optional[str], int]:
    if not habits:
        return (None, 0)
    # find habit with max streak
    def better(a, b):
        return a if longest_streak_for(a) >= longest_streak_for(b) else b

    max_h = reduce(better, habits)
    return (max_h.name, longest_streak_for(max_h))

"""fixtures.py
Provide 5 predefined habits and populate 4 weeks of example completion events.
"""

from datetime import datetime, timedelta
from habit import Habit


def init_fixtures(manager):
    # Predefined set
    predefined = [
        ("Brush Teeth", "daily"),
        ("Exercise", "daily"),
        ("Read", "daily"),
        ("Grocery Shopping", "weekly"),
        ("Call Family", "weekly"),
    ]

    # remove existing with same names
    for name, _ in predefined:
        try:
            manager.delete_habit(name)
        except Exception:
            pass

    for name, period in predefined:
        h = manager.create_habit(name, period)
        # populate last 28 days
        today = datetime.now().date()
        if period == "daily":
            # mark many days but leave some misses: simulate 80% completion
            for offset in range(0, 28):
                # skip every 6th day to simulate a miss
                if offset % 6 != 0:
                    dt = datetime.combine(today - timedelta(days=offset), datetime.min.time())
                    h.complete(dt)
        else:  # weekly
            # mark one day per week (e.g., every Sunday)
            for week_offset in range(0, 4):
                dt = datetime.combine(today - timedelta(weeks=week_offset), datetime.min.time())
                h.complete(dt)
        manager.storage.save_habit(h)

"""cli.py
Minimal command-line interface to interact with HabitManager.
"""

import argparse
from manager import HabitManager
from storage import Storage
import analytics as analytics_mod


def main():
    parser = argparse.ArgumentParser(description="Habit Tracker CLI")
    parser.add_argument("action", choices=["create", "complete", "list", "streak", "delete", "init-fixtures"], help="action")
    parser.add_argument("--name", help="habit name")
    parser.add_argument("--period", choices=["daily", "weekly"], help="periodicity")
    args = parser.parse_args()

    storage = Storage()
    manager = HabitManager(storage)

    if args.action == "create":
        if not args.name or not args.period:
            print("create requires --name and --period")
            return
        manager.create_habit(args.name, args.period)
        print(f"Habit created: {args.name} ({args.period})")

    elif args.action == "complete":
        if not args.name:
            print("complete requires --name")
            return
        try:
            manager.complete_habit(args.name)
            print(f"Habit completed: {args.name}")
        except ValueError as e:
            print(str(e))

    elif args.action == "list":
        habits = manager.list_habits()
        if not habits:
            print("No habits yet.")
            return
        for h in habits:
            print(f"- {h.name} ({h.periodicity}) created: {h.created_at.date()} completions: {len(h.completions)}")

    elif args.action == "streak":
        if args.name:
            h = manager.find(args.name)
            if not h:
                print("habit not found")
                return
            print(f"{h.name} current streak: {h.get_streak()}")
        else:
            name, val = analytics_mod.longest_streak_all(manager.list_habits())
            if name is None:
                print("No habits found")
            else:
                print(f"Longest streak overall: {name} = {val}")

    elif args.action == "delete":
        if not args.name:
            print("delete requires --name")
            return
        ok = manager.delete_habit(args.name)
        print("deleted" if ok else "not found")

    elif args.action == "init-fixtures":
        from fixtures import init_fixtures

        init_fixtures(manager)
        print("fixtures loaded")


if __name__ == "__main__":
    main()

"""Unit tests for Habit class streak logic."""

from habit import Habit
from datetime import datetime, timedelta


def test_daily_streak_ok():
    h = Habit("Daily", "daily")
    today = datetime.now().date()
    for i in range(3):
        h.complete(datetime.combine(today - timedelta(days=i), datetime.min.time()))
    assert h.get_streak() == 3


def test_daily_streak_broken():
    h = Habit("DailyBroken", "daily")
    today = datetime.now().date()
    h.complete(datetime.combine(today - timedelta(days=1), datetime.min.time()))
    assert h.get_streak() == 0


def test_weekly_streak():
    h = Habit("Weekly", "weekly")
    now = datetime.now()
    h.complete(now)
    h.complete(now - timedelta(weeks=1))
    assert h.get_streak() >= 2

# Habit Tracker Backend

## 📘 Overview
A simple Python backend for tracking and analyzing daily and weekly habits.  
Users can create habits, mark completions, view streaks, and analyze progress — all via a clean command-line interface (CLI).

---

## ⚙️ Installation

### Requirements
- Python ≥ 3.7
- pip
- (Optional) Virtual environment

### Steps
```bash
git clone <your_repo> habit_tracker_project
cd habit_tracker_project

python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate

pip install -r requirements.txt
