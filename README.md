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

    
