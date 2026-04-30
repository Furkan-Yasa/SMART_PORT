# ============================================================
# SMART PORT - PRIORITY ZONE VERSION
# 16x16 | PORT=2 | YARD=9 | ENTRY=5
# RED -> LEFT 3 COLS
# YELLOW -> MID 3 COLS
# GREEN -> RIGHT 3 COLS
# V CRANE MOVES STEP BY STEP
# ============================================================

import numpy as np
import random
import matplotlib.pyplot as plt
import matplotlib.patches as patches
import imageio
from IPython.display import Image, display

# ============================================================
# CONFIG
# ============================================================

GRID_SIZE = 16

PORT_COLS  = 2
YARD_COLS  = 9
ENTRY_COLS = 5

PORT_END    = PORT_COLS
YARD_START  = PORT_END
YARD_END    = PORT_END + YARD_COLS
ENTRY_START = YARD_END

FPS = 3

# ------------------------------------------------------------
# PRIORITY ZONES (YARD)
# ------------------------------------------------------------
# RED    -> cols 2,3,4
# YELLOW -> cols 5,6,7
# GREEN  -> cols 8,9,10

RED_ZONE    = range(YARD_START, YARD_START + 3)
YELLOW_ZONE = range(YARD_START + 3, YARD_START + 6)
GREEN_ZONE  = range(YARD_START + 6, YARD_END)

# ------------------------------------------------------------
# CONTAINERS
# ------------------------------------------------------------

SHAPES = [(1,2),(2,1),(1,3),(3,1)]
PRIORITIES = ["R","Y","G"]   # Red Yellow Green
WEIGHTS = ["L","M","H"]      # Light Medium Heavy

# ============================================================
# RANDOM CONTAINER
# ============================================================

def random_container():
    return {
        "shape": random.choice(SHAPES),
        "priority": random.choice(PRIORITIES),
        "weight": random.choice(WEIGHTS)
    }

# ============================================================
# ENVIRONMENT
# ============================================================

class SmartPort:

    def __init__(self):
        self.reset()

    # --------------------------------------------------------
    # RESET
    # --------------------------------------------------------
    def reset(self):

        self.grid = [["." for _ in range(GRID_SIZE)] for _ in range(GRID_SIZE)]
        self.text = [["" for _ in range(GRID_SIZE)] for _ in range(GRID_SIZE)]

        # PORT
        for r in range(GRID_SIZE):
            for c in range(PORT_COLS):
                self.grid[r][c] = "~"

        # Crane start
        self.vx = 8
        self.vy = YARD_END - 1
        self.holding = None

        self.queue = []
        self.entry_positions = []

        self.step_count = 0
        self.total_reward = 0

        self.generate_full_entry()

    # --------------------------------------------------------
    # ENTRY AREA FULL RANDOM
    # --------------------------------------------------------
    def generate_full_entry(self):

        occupied = np.zeros((GRID_SIZE, ENTRY_COLS), dtype=int)

        for r in range(GRID_SIZE):
            for c in range(ENTRY_COLS):

                if occupied[r][c] == 1:
                    continue

                placed = False

                for _ in range(30):

                    cont = random_container()
                    h,w = cont["shape"]

                    if r+h <= GRID_SIZE and c+w <= ENTRY_COLS:

                        ok = True

                        for i in range(h):
                            for j in range(w):
                                if occupied[r+i][c+j] == 1:
                                    ok = False

                        if ok:

                            for i in range(h):
                                for j in range(w):

                                    occupied[r+i][c+j] = 1

                                    rr = r+i
                                    cc = ENTRY_START + c + j

                                    self.grid[rr][cc] = cont["priority"]

                                    if i == 0 and j == 0:
                                        self.text[rr][cc] = cont["weight"]

                            self.queue.append(cont)
                            self.entry_positions.append((r, ENTRY_START + c))

                            placed = True
                            break

                if placed:
                    continue

    # --------------------------------------------------------
    # CLEAR ENTRY CONTAINER
    # --------------------------------------------------------
    def clear_entry_container(self, idx):

        cont = self.queue[idx]
        r,c = self.entry_positions[idx]

        h,w = cont["shape"]

        for i in range(h):
            for j in range(w):
                self.grid[r+i][c+j] = "."
                self.text[r+i][c+j] = ""

    # --------------------------------------------------------
    # PRIORITY BASED SLOT SEARCH
    # --------------------------------------------------------
    def get_zone_cols(self, priority):

        if priority == "R":
            return RED_ZONE

        elif priority == "Y":
            return YELLOW_ZONE

        else:
            return GREEN_ZONE

    def find_yard_slot(self, cont):

        h,w = cont["shape"]
        zone_cols = self.get_zone_cols(cont["priority"])

        # First try correct zone
        for r in range(GRID_SIZE):
            for c in zone_cols:

                if c + w > YARD_END:
                    continue

                ok = True

                for i in range(h):
                    for j in range(w):
                        if r+i >= GRID_SIZE:
                            ok = False
                        elif self.grid[r+i][c+j] != ".":
                            ok = False

                if ok:
                    return (r,c)

        # Fallback any yard slot
        for r in range(GRID_SIZE):
            for c in range(YARD_START, YARD_END):

                if c + w > YARD_END:
                    continue

                ok = True

                for i in range(h):
                    for j in range(w):
                        if r+i >= GRID_SIZE:
                            ok = False
                        elif self.grid[r+i][c+j] != ".":
                            ok = False

                if ok:
                    return (r,c)

        return None

    # --------------------------------------------------------
    # MOVE ONE STEP
    # --------------------------------------------------------
    def move_to(self, tx, ty):

        if self.vx < tx:
            self.vx += 1
        elif self.vx > tx:
            self.vx -= 1
        elif self.vy < ty:
            self.vy += 1
        elif self.vy > ty:
            self.vy -= 1

        self.step_count += 1

    # --------------------------------------------------------
    # PICKUP
    # --------------------------------------------------------
    def pickup_first(self):

        if len(self.queue) == 0:
            return False

        cont = self.queue[0]

        self.clear_entry_container(0)

        self.queue.pop(0)
        self.entry_positions.pop(0)

        self.holding = cont
        self.total_reward += 10

        return True

    # --------------------------------------------------------
    # DROPOFF
    # --------------------------------------------------------
    def dropoff(self):

        if self.holding is None:
            return False

        pos = self.find_yard_slot(self.holding)

        if pos is None:
            self.total_reward -= 20
            return False

        r,c = pos
        h,w = self.holding["shape"]

        for i in range(h):
            for j in range(w):

                self.grid[r+i][c+j] = self.holding["priority"]

                if i == 0 and j == 0:
                    self.text[r+i][c+j] = self.holding["weight"]

        self.holding = None
        self.total_reward += 25

        return True

    # --------------------------------------------------------
    # RENDER
    # --------------------------------------------------------
    def render(self):

        fig, ax = plt.subplots(figsize=(8,8))

        colors = {
            ".":"white",
            "~":"skyblue",
            "R":"red",
            "Y":"yellow",
            "G":"green"
        }

        for r in range(GRID_SIZE):
            for c in range(GRID_SIZE):

                rect = patches.Rectangle(
                    (c, GRID_SIZE-r-1),
                    1,1,
                    facecolor=colors.get(self.grid[r][c], "white"),
                    edgecolor="gray"
                )
                ax.add_patch(rect)

                if self.text[r][c] != "":
                    ax.text(
                        c+0.5,
                        GRID_SIZE-r-0.5,
                        self.text[r][c],
                        ha="center",
                        va="center",
                        fontsize=7,
                        weight="bold"
                    )

        # Yard zone lines
        ax.axvline(PORT_END, linewidth=3, color="blue")
        ax.axvline(YARD_START + 3, linewidth=2, linestyle="--")
        ax.axvline(YARD_START + 6, linewidth=2, linestyle="--")
        ax.axvline(ENTRY_START, linewidth=3, color="black")

        # Crane V
        ax.add_patch(
            patches.Circle(
                (self.vy+0.5, GRID_SIZE-self.vx-0.5),
                0.30,
                color="black"
            )
        )

        ax.text(
            self.vy+0.5,
            GRID_SIZE-self.vx-0.5,
            "V",
            color="white",
            ha="center",
            va="center",
            fontsize=8,
            weight="bold"
        )

        plt.title(f"STEP:{self.step_count} | REWARD:{self.total_reward}")

        ax.set_xlim(0, GRID_SIZE)
        ax.set_ylim(0, GRID_SIZE)
        ax.set_xticks([])
        ax.set_yticks([])

        fig.tight_layout()
        fig.canvas.draw()

        img = np.asarray(fig.canvas.buffer_rgba())
        plt.close(fig)

        return img

# ============================================================
# RUN
# ============================================================

env = SmartPort()

frames = []

while len(env.queue) > 0:

    er, ec = env.entry_positions[0]

    # Move to container
    while (env.vx, env.vy) != (er, ec):
        frames.append(env.render())
        env.move_to(er, ec)

    frames.append(env.render())
    env.pickup_first()

    # Find target slot
    pos = env.find_yard_slot(env.holding)

    if pos is None:
        break

    tr, tc = pos

    # Move to yard
    while (env.vx, env.vy) != (tr, tc):
        frames.append(env.render())
        env.move_to(tr, tc)

    frames.append(env.render())
    env.dropoff()

frames.append(env.render())

imageio.mimsave("smart_port_priority.gif", frames, fps=FPS)

print("GIF oluşturuldu 🎬 -> smart_port_priority.gif")

display(Image(filename="smart_port_priority.gif"))
