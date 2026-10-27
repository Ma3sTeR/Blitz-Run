import asyncio
import inspect
import math
import random

import flet as ft
import flet.canvas as cv

W, H, GY = 800, 400, 330  # logical game size and ground height
BOSS_FIRST = 1200  # score of the first boss
BOSS_EVERY = 1500  # score gap between bosses


try:
    STROKE_STYLE = ft.PaintingStyle.STROKE
except Exception:
    STROKE_STYLE = None
OVAL = getattr(cv, "Oval", None)


def hex2rgb(h):
    return int(h[1:3], 16), int(h[3:5], 16), int(h[5:7], 16)


def mix(a, b, t):
    ra, ga, ba = hex2rgb(a)
    rb, gb, bb = hex2rgb(b)
    return "#%02x%02x%02x" % (int(ra + (rb - ra) * t), int(ga + (gb - ga) * t), int(ba + (bb - ba) * t))


def gradient_stops(stops, n):
    out = []
    for i in range(n):
        pos = i / (n - 1) * (len(stops) - 1)
        k = min(int(pos), len(stops) - 2)
        out.append(mix(stops[k], stops[k + 1], pos - k))
    return out


# one theme (zone) per boss defeated: sunset -> day -> night -> ...
THEMES = [
    dict(sky=["#1b2a8a", "#5b3fa0", "#ff7a59"], orb=(595, 95, 38, "#ffd23f"), glow="#ffb347", moon=False,
         stars=False, mtn_far="#4a3fb0", mtn="#3a2f9e", snow="#ff9ab0", bush="#2fa84f", grass="#46d06a",
         grass_d="#2a7a3e", grass_l="#7dffa0", dirt="#8d5826", dirt2="#6a3f17", cloud="#55ffc0b0"),
    dict(sky=["#3c8cf0", "#8cc8ff", "#d8f0ff"], orb=(650, 70, 30, "#fff3a0"), glow="#ffe680", moon=False,
         stars=False, mtn_far="#8fb0d8", mtn="#6c8ebf", snow="#f2f8ff", bush="#35b857", grass="#5be37a",
         grass_d="#2f9a4a", grass_l="#a8ffbf", dirt="#a06a30", dirt2="#855324", cloud="#d8ffffff"),
    dict(sky=["#05082a", "#1a1f5e", "#3d2a7a"], orb=(600, 80, 28, "#f2f2ff"), glow="#8890ff", moon=True,
         stars=True, mtn_far="#232672", mtn="#1a1a5a", snow="#aab4ff", bush="#146a58", grass="#1fa88a",
         grass_d="#116655", grass_l="#6affd8", dirt="#4a3a5a", dirt2="#3a2d4a", cloud="#22aab4ff"),
]


class Game:
    def __init__(self):
        self.hi = 0
        self.state = "start"
        self.t = 0
        self.save_pending = False
        self.new_hi = False
        self.hq = True
        self.reset()

    def reset(self):
        self.px, self.py, self.pw, self.ph = 120, GY - 34, 30, 34
        self.vy = 0.0
        self.on = True
        self.jumps = 0
        self.held = False
        self.last_surf = GY
        self.obs, self.rings, self.floaters, self.parts = [], [], [], []
        self.speed, self.dist, self.score, self.bonus = 6.0, 0.0, 0, 0
        self.lives, self.inv, self.shake = 3, 0, 0
        self.next_obs, self.next_ring = 60, 30
        self.boss = None
        self.boss_level = 0
        self.next_boss = BOSS_FIRST
        self.ring_count = 0
        self.banner = 0

    # ---------- input ----------
    def jump(self):
        if self.state != "play":
            self.reset()
            self.state = "play"
            return
        if self.on:
            self.vy, self.on, self.jumps, self.held = -13.5, False, 1, True
        elif self.jumps < 2:
            self.vy, self.jumps, self.held = -11.0, 2, True

    def release(self):
        self.held = False
        if self.vy < -5:
            self.vy = -5

    # ---------- helpers ----------
    def add_bonus(self, n):
        self.bonus += n

    def burst(self, x, y, n, color, power=4):
        for _ in range(n):
            self.parts.append(dict(x=x, y=y, vx=random.uniform(-power, power),
                                   vy=-random.random() * power * 1.5, l=random.randint(20, 35), c=color))

    def surface(self):
        """Y of the ground right under the player (ramps raise it)."""
        fx = self.px + 15
        best = 0.0
        for o in self.obs:
            if o["k"] == "ramp" and o["x"] <= fx <= o["x"] + o["w"]:
                best = max(best, o["h"] * (fx - o["x"]) / o["w"])
        return GY - best

    # ---------- spawning ----------
    def spawn(self):
        r = random.random()
        gap_extra = 0
        if r < 0.28:
            self.obs.append(dict(k="spike", x=W + 20, y=GY - 24, w=34, h=24, dead=0))
        elif r < 0.50:
            self.obs.append(dict(k="bot", x=W + 20, y=GY - 34, w=34, h=34, dead=0))
        elif r < 0.62:
            self.obs.append(dict(k="spike", x=W + 20, y=GY - 24, w=34, h=24, dead=0))
            self.obs.append(dict(k="spike", x=W + 54, y=GY - 24, w=34, h=24, dead=0))
        elif r < 0.78:
            sx = W + 20
            self.obs.append(dict(k="spring", x=sx, y=GY - 14, w=34, h=14, dead=0, used=0))
            # trail of rings along the spring's flight path
            for f in (12, 18, 24, 30, 36):
                h = 19 * f - 0.425 * f * f
                self.rings.append(dict(x=sx + 17 + self.speed * f, y=GY - 17 - h))
            gap_extra = 40
        else:
            self.obs.append(dict(k="ramp", x=W + 20, y=GY - 46, w=130, h=46, dead=0))
            gap_extra = 60
        self.next_obs = max(38, (70 + random.random() * 60) * (8 / self.speed) + 30) + gap_extra

    def spawn_rings(self):
        n = 4 + int(random.random() * 3)
        hy = GY - 50 - random.random() * 90
        for i in range(n):
            y = hy - math.sin(i / (n - 1) * math.pi) * 34
            self.rings.append(dict(x=W + 20 + i * 34, y=y))
        self.next_ring = 90 + random.random() * 80

    def start_boss(self):
        hp = min(7, 4 + self.boss_level)
        self.boss = dict(x=W + 100, y=GY - 84, w=84, h=84, hp=hp, max=hp, phase="enter", timer=0, imm=0)

    # ---------- damage ----------
    def hit(self):
        if self.inv > 0:
            return
        self.lives -= 1
        self.inv, self.shake = 100, 12
        self.burst(self.px + 15, self.py + 15, 10, "#ffd23f")
        if self.lives <= 0:
            self.state = "over"
            self.new_hi = self.score > self.hi
            if self.new_hi:
                self.hi = self.score
                self.save_pending = True

    # ---------- boss ----------
    def update_boss(self):
        b = self.boss
        if b["imm"] > 0:
            b["imm"] -= 1
        ph = b["phase"]
        if ph == "enter":
            b["x"] -= 6
            if b["x"] <= 620:
                b["phase"], b["timer"] = "wait", 60
        elif ph == "wait":
            b["timer"] -= 1
            if b["timer"] <= 0:
                b["phase"] = "charge"
        elif ph == "charge":
            b["x"] -= 10 + (b["max"] - b["hp"]) * 1.5
            if b["x"] < -120:
                b["x"], b["phase"] = W + 100, "enter"

        if b["imm"] > 0 or self.state != "play":
            return
        bx, by, bw, bh = b["x"] + 8, b["y"] + 8, b["w"] - 16, b["h"] - 8
        x, y, w, h = self.px + 4, self.py + 4, self.pw - 8, self.ph - 6
        if x < bx + bw and x + w > bx and y < by + bh and y + h > by:
            if (not self.on) and self.vy > 0 and self.py + self.ph <= b["y"] + 34:
                b["hp"] -= 1
                b["imm"] = 40
                self.vy, self.jumps = -11.0, 1
                self.burst(b["x"] + 40, b["y"] + 10, 12, "#b04dff")
                self.floaters.append(dict(x=b["x"] + 20, y=b["y"] - 6, t="HIT!", l=40))
                if b["hp"] <= 0:
                    self.burst(b["x"] + 42, b["y"] + 42, 45, "#ffd23f", 7)
                    self.burst(b["x"] + 42, b["y"] + 42, 25, "#ff4d6d", 6)
                    self.add_bonus(500)
                    self.floaters.append(dict(x=b["x"], y=b["y"], t="+500", l=60))
                    self.shake = 20
                    self.boss = None
                    self.boss_level += 1
                    self.banner = 130
                    self.next_boss = int(self.dist / 12) + self.bonus + BOSS_EVERY
            else:
                self.hit()

    # ---------- main update ----------
    def update(self):
        self.t += 1
        if self.state != "play":
            return
        self.speed = min(14.0, 6 + self.dist / 2500)
        self.dist += self.speed
        self.score = int(self.dist / 12) + self.bonus

        if self.boss is None and self.score >= self.next_boss:
            self.start_boss()

        # player physics (with ramps)
        surf = self.surface()
        if self.on:
            if self.last_surf < GY - 10 and surf >= GY - 0.5:
                # ran off the end of a ramp -> launch
                self.on, self.vy, self.jumps = False, -8.0, 1
                self.py += self.vy
            else:
                self.py = surf - self.ph
                self.vy = 0.0
        else:
            self.vy += 0.55 if (self.held and self.vy < 0) else 0.85
            self.py += self.vy
            if self.vy >= 0 and self.py + self.ph >= surf:
                self.py, self.vy, self.on, self.jumps = surf - self.ph, 0.0, True, 0
        self.last_surf = surf

        self.inv = max(0, self.inv - 1)
        self.shake = max(0, self.shake - 1)
        self.banner = max(0, self.banner - 1)
        if self.hq and self.on and self.t % 4 == 0 and self.speed > 7:
            self.parts.append(dict(k="d", x=self.px + 6, y=self.py + self.ph - 2, vx=-2.0, vy=-0.4, l=16,
                                   c="#88ffffff"))

        if self.boss is None:
            self.next_obs -= 1
            if self.next_obs <= 0:
                self.spawn()
        self.next_ring -= 1
        if self.next_ring <= 0:
            self.spawn_rings()

        airborne = not self.on
        for o in self.obs:
            k = o["k"]
            o["x"] -= self.speed + (1.5 if k == "bot" and not o["dead"] else 0)
            if k == "ramp":
                continue
            if k == "spring":
                if o["used"]:
                    o["used"] += 1
                elif (self.px + 4 < o["x"] + o["w"] and self.px + self.pw - 4 > o["x"]
                      and self.py + self.ph >= o["y"] - 2 and (self.on or self.vy >= 0)):
                    o["used"] = 1
                    self.vy, self.on, self.jumps, self.held = -19.0, False, 1, False
                    self.floaters.append(dict(x=o["x"], y=o["y"] - 20, t="BOING!", l=35))
                continue
            if o["dead"]:
                o["dead"] += 1
                o["y"] += 3
                continue
            x, y, w, h = self.px + 4, self.py + 4, self.pw - 8, self.ph - 6
            if x < o["x"] + o["w"] - 4 and x + w > o["x"] + 4 and y < o["y"] + o["h"] and y + h > o["y"] + 4:
                if k == "bot" and airborne and self.vy > 0:
                    o["dead"] = 1
                    self.vy, self.jumps = -9.0, 1
                    self.add_bonus(50)
                    self.floaters.append(dict(x=o["x"], y=o["y"], t="+50", l=40))
                    self.burst(o["x"] + 17, o["y"] + 17, 8, "#ff4d6d", 3)
                else:
                    self.hit()
        self.obs = [o for o in self.obs if o["x"] > -150 and o["dead"] < 30]

        if self.boss is not None:
            self.update_boss()

        keep = []
        for r in self.rings:
            r["x"] -= self.speed
            if abs(r["x"] - (self.px + 15)) < 20 and abs(r["y"] - (self.py + 17)) < 24:
                self.add_bonus(10)
                self.ring_count += 1
                self.floaters.append(dict(x=r["x"], y=r["y"], t="+10", l=30))
            elif r["x"] > -20:
                keep.append(r)
        self.rings = keep

        for f in self.floaters:
            f["y"] -= 1
            f["l"] -= 1
        self.floaters = [f for f in self.floaters if f["l"] > 0]
        for p in self.parts:
            p["x"] += p["vx"]
            p["y"] += p["vy"]
            if p.get("k") != "d":
                p["vy"] += 0.4
            p["l"] -= 1
        self.parts = [p for p in self.parts if p["l"] > 0]


def main(page: ft.Page):
    page.title = "Blitz Run"
    page.bgcolor = "#0b1030"
    page.padding = 8
    page.horizontal_alignment = ft.CrossAxisAlignment.CENTER

    g = Game()
    avail = (page.width or 420) - 16
    K = max(0.3, min(1.2, avail / W))
    canvas = cv.Canvas(width=W * K, height=H * K, shapes=[])

    # ---------- drawing helpers (all coordinates in logical units) ----------
    def P(c, stroke=0):
        if stroke and STROKE_STYLE is not None:
            return ft.Paint(color=c, style=STROKE_STYLE, stroke_width=stroke * K)
        return ft.Paint(color=c)

    def R(x, y, w, h, c):
        return cv.Rect(x * K, y * K, w * K, h * K, paint=P(c))

    def C(x, y, r, c, stroke=0):
        return cv.Circle(x * K, y * K, r * K, paint=P(c, stroke))

    def O(x, y, w, h, c, stroke=0):
        if OVAL is None:
            return cv.Rect(x * K, y * K, w * K, h * K, paint=P(c, stroke))
        return OVAL(x * K, y * K, w * K, h * K, paint=P(c, stroke))

    def poly(pts, c):
        els = [cv.Path.MoveTo(pts[0][0] * K, pts[0][1] * K)]
        els += [cv.Path.LineTo(x * K, y * K) for x, y in pts[1:]]
        els.append(cv.Path.Close())
        return cv.Path(els, paint=P(c))

    def T(s, x, y, size, c, center=False):
        if center:
            x -= len(s) * size * 0.3
        return cv.Text(x * K, y * K, s, style=ft.TextStyle(size=size * K, color=c, weight=ft.FontWeight.BOLD))

    def TS(s, x, y, size, c, center=False, sh="#000000"):
        """text with drop shadow"""
        return [T(s, x + 2, y + 2, size, sh, center), T(s, x, y, size, c, center)]

    # ---------- static scenery, built once per theme and reused ----------
    rnd = random.Random(7)
    STARS = [(rnd.randint(10, W - 10), rnd.randint(8, 190), rnd.choice((1.2, 1.6, 2.2))) for _ in range(26)]
    for th in THEMES:
        sky = []
        n = 14
        cols = gradient_stops(th["sky"], n)
        for i in range(n):
            sky.append(R(0, i * GY / n, W, GY / n + 1.5, cols[i]))
        ox_, oy_, orr, oc = th["orb"]
        sky += [C(ox_, oy_, orr * 2.4, mix(cols[3], th["glow"], 0.18)),
                C(ox_, oy_, orr * 1.7, mix(cols[3], th["glow"], 0.35)),
                C(ox_, oy_, orr * 1.3, mix(cols[3], th["glow"], 0.6)),
                C(ox_, oy_, orr, oc)]
        if th["moon"]:
            sky += [C(ox_ - 8, oy_ - 6, 6, "#c9c9ee"), C(ox_ + 10, oy_ + 8, 4, "#c9c9ee"),
                    C(ox_ + 6, oy_ - 12, 3, "#c9c9ee")]
        th["sky_s"] = sky
        th["stars_s"] = [C(x, y, r, "#ffffff") for x, y, r in STARS] if th["stars"] else []
        th["gnd_s"] = [R(0, GY, W, H - GY, th["dirt"])]
        th["grass_s"] = [R(0, GY, W, 10, th["grass"]), R(0, GY + 10, W, 4, th["grass_d"]),
                         R(0, GY, W, 3, th["grass_l"])]

    def draw_boss(s, b, ox):
        x, y = b["x"] + ox, b["y"]
        blink = b["imm"] > 0 and (g.t >> 1) & 1
        body = "#ffffff" if blink else "#7a2fff"
        body_d = "#ffffff" if blink else "#5a1fd0"
        head = "#ffffff" if blink else "#b04dff"
        angry = b["phase"] == "wait"
        eye = "#ff2020" if angry else "#111111"
        if g.hq:
            s.append(C(x + 42, y + 44, 74, "#22b04dff"))
            s.append(C(x + 42, y + 44, 58, "#22b04dff"))
        s += [R(x + 4, y + 68, 28, 16, "#1c1c38"), R(x + 52, y + 68, 28, 16, "#1c1c38"),   # treads
              R(x + 8, y + 72, 20, 4, "#555577"), R(x + 56, y + 72, 20, 4, "#555577"),
              R(x - 2, y + 28, 88, 48, "#2a0f66"),                                          # outline
              R(x, y + 30, 84, 44, body), R(x, y + 56, 84, 18, body_d),
              R(x + 8, y - 2, 68, 38, "#2a0f66"), R(x + 10, y, 64, 34, head),               # head
              R(x + 12, y + 2, 60, 6, "#d28aff" if not blink else "#ffffff"),
              R(x + 38, y - 14, 6, 14, "#aaaaaa"), C(x + 41, y - 17, 6, "#ff4d6d"),
              C(x + 41, y - 17, 10, "#33ff4d6d"),
              R(x + 17, y + 9, 20, 16, "#ffffff"), R(x + 47, y + 9, 20, 16, "#ffffff"),
              R(x + 24, y + 12, 9, 10, eye), R(x + 54, y + 12, 9, 10, eye),
              R(x + 26, y + 13, 3, 3, "#ffffff"), R(x + 56, y + 13, 3, 3, "#ffffff"),
              poly([(x + 14, y + 4), (x + 38, y + 14), (x + 38, y + 9)], "#111111"),
              poly([(x + 70, y + 4), (x + 46, y + 14), (x + 46, y + 9)], "#111111"),
              R(x + 20, y + 42, 44, 12, "#2a0f66"),
              R(x + 24, y + 45, 8, 6, "#ffffff"), R(x + 38, y + 45, 8, 6, "#ffffff"), R(x + 52, y + 45, 8, 6, "#ffffff"),
              C(x + 8, y + 62, 3, "#ffd23f"), C(x + 76, y + 62, 3, "#ffd23f"),
              R(x + 30, y + 60, 24, 8, "#ffd23f")]
        if angry and (g.t >> 2) & 1:
            s += TS("!", x + 36, y - 56, 36, "#ff4d6d")
        bw = 240
        s += [R(W / 2 - bw / 2 - 4, 10, bw + 8, 20, "#000000"), R(W / 2 - bw / 2, 14, bw, 12, "#3a1a4a"),
              R(W / 2 - bw / 2, 14, bw * b["hp"] / b["max"], 12, "#ff4d6d"),
              R(W / 2 - bw / 2, 14, bw * b["hp"] / b["max"], 4, "#ff9ab0")]

    def draw_player(s, ox):
        t, hq = g.t, g.hq
        if g.inv > 0 and (t >> 2) & 1:
            return
        x, y = g.px + ox, g.py
        if not g.on:
            cx, cy = x + 15, y + 17
            if hq:
                s += [C(cx, cy, 26, "#222b6bff"), C(cx - g.speed * 1.4, cy, 14, "#332b6bff"),
                      C(cx - g.speed * 2.8, cy, 10, "#222b6bff")]
            for i in range(6):
                a = t * 0.4 + i * math.pi / 3
                ca, sa = math.cos(a), math.sin(a)
                s.append(poly([(cx + ca * 12 - sa * 5, cy + sa * 12 + ca * 5),
                               (cx + ca * 25, cy + sa * 25),
                               (cx + ca * 12 + sa * 5, cy + sa * 12 - ca * 5)], "#1a45b8" if i % 2 else "#2a5be0"))
            s += [C(cx, cy, 17, "#0a1a5a"), C(cx, cy, 16, "#2b6bff"), C(cx, cy, 10, "#9cc0ff", 3),
                  C(cx - 6, cy - 7, 4, "#ccffffff")]
            return
        f = (t >> 2) & 1
        arm = 2 if f else -2
        s += [poly([(x + 4, y + 3), (x - 17, y - 3), (x + 4, y + 12)], "#2a5be0"),
              poly([(x + 4, y + 12), (x - 19, y + 12), (x + 4, y + 21)], "#1a45b8"),
              poly([(x + 5, y + 20), (x - 15, y + 27), (x + 7, y + 28)], "#2a5be0"),
              poly([(x + 5, y + 3), (x + 9, y - 7), (x + 15, y + 3)], "#2b6bff"),             # ear
              R(x + 1, y + 1, 28, 25, "#0a1a5a"), R(x + 2, y + 2, 26, 23, "#2b6bff"),         # outline + body
              R(x + 4, y + 4, 12, 5, "#6a94ff"), R(x + 6, y + 21, 14, 8, "#f2c9a0"),          # highlight + belly
              R(x + 14, y + 11, 14, 11, "#f2c9a0"),                                           # muzzle
              R(x + 16, y + 4, 11, 11, "#ffffff"), R(x + 22, y + 6, 4, 8, "#111111"),
              R(x + 22, y + 6, 2, 2, "#ffffff"),
              R(x + 26, y + 13, 6, 5, "#111111"),
              R(x + 10, y + 19 + arm, 11, 5, "#f2c9a0"),                                      # arm
              R(x + (1 if f else 12), y + 28, 13, 6, "#ff4d6d"), R(x + (14 if f else 3), y + 28, 13, 6, "#ff4d6d"),
              R(x + (1 if f else 12), y + 30, 13, 2, "#ffffff"), R(x + (14 if f else 3), y + 30, 13, 2, "#ffffff"),
              R(x + (1 if f else 12), y + 33, 13, 2, "#7a1a2a"), R(x + (14 if f else 3), y + 33, 13, 2, "#7a1a2a")]
        if g.speed > 10 and hq:
            for k2 in range(1, 4):
                s.append(R(x - 18 - k2 * 14, y + 6 + k2 * 6, 14, 2, "#88c8d2ff"))

    def draw():
        t, hq = g.t, g.hq
        th = THEMES[g.boss_level % 3]
        s = list(th["sky_s"])
        ox = random.uniform(-3, 3) if g.shake else 0

        if hq:
            s += th["stars_s"]
            for i in range(3):
                cx = (i * 330 + 120 - g.dist * 0.04) % (W + 240) - 120
                cy = 58 + i * 30
                s += [C(cx, cy, 16, th["cloud"]), C(cx + 20, cy - 8, 21, th["cloud"]), C(cx + 44, cy, 15, th["cloud"])]
            o0 = (g.dist * 0.07) % 400
            for i in range(0, 4):
                bx = i * 400 - o0
                s.append(poly([(bx - 40, GY), (bx + 150, GY - 150), (bx + 340, GY)], th["mtn_far"]))
        o = (g.dist * 0.15) % 400
        for i in range(0, 3):
            bx = i * 400 - o
            s.append(poly([(bx, GY), (bx + 120, GY - 120), (bx + 250, GY)], th["mtn"]))
            s.append(poly([(bx + 160, GY), (bx + 290, GY - 80), (bx + 400, GY)], th["mtn"]))
            if hq:
                s.append(poly([(bx + 90, GY - 90), (bx + 120, GY - 120), (bx + 150, GY - 90)], th["snow"]))
        o2 = (g.dist * 0.4) % 200
        for i in range(0, 5):
            cx = i * 200 - o2
            s += [R(cx, GY - 24, 110, 24, th["bush"]), C(cx + 28, GY - 26, 24, th["bush"]),
                  C(cx + 60, GY - 32, 30, th["bush"]), C(cx + 88, GY - 25, 21, th["bush"])]
        s += th["gnd_s"]
        o3 = g.dist % 160
        for i in range(0, 6):
            s.append(R(i * 160 - o3 + 80, GY, 80, H - GY, th["dirt2"]))
        s += th["grass_s"]
        if hq:
            o4 = g.dist % 80
            for i in range(0, 11):
                gx = i * 80 - o4
                s.append(poly([(gx, GY), (gx + 6, GY - 9), (gx + 12, GY)], th["grass_l"]))

        # ramps (behind everything else)
        for ob in g.obs:
            if ob["k"] == "ramp":
                x = ob["x"]
                s += [poly([(x, GY), (x + ob["w"], GY), (x + ob["w"], GY - ob["h"])], th["dirt"]),
                      poly([(x + 40, GY), (x + ob["w"], GY), (x + ob["w"], GY - ob["h"] + 30)], th["dirt2"]),
                      poly([(x, GY), (x + ob["w"], GY - ob["h"]), (x + ob["w"], GY - ob["h"] + 10),
                            (x + 4, GY + 4)], th["grass"])]

        # rings
        for r in g.rings:
            w = abs(math.cos(t / 8 + r["x"] / 40)) * 9 + 2
            if hq:
                s.append(C(r["x"], r["y"], 15, "#22ffd23f"))
            s.append(O(r["x"] - w, r["y"] - 11, w * 2, 22, "#ffd23f", 4))
            if hq and w > 6:
                s.append(C(r["x"] - w + 3, r["y"] - 6, 2, "#ffffff"))

        # springs, spikes, bots
        for ob in g.obs:
            k = ob["k"]
            x, y = ob["x"] + ox, ob["y"]
            if k == "spike":
                for off in (0, 17):
                    s += [poly([(x + off, y + 24), (x + off + 8.5, y), (x + off + 8.5, y + 24)], "#eceef9"),
                          poly([(x + off + 8.5, y + 24), (x + off + 8.5, y), (x + off + 17, y + 24)], "#9ea2bd")]
                s.append(R(x, y + 20, 34, 4, "#5b5f7a"))
            elif k == "spring":
                hh = 4 if 0 < ob["used"] < 8 else (16 if ob["used"] else 9)
                s += [R(x + 2, GY - 5, 30, 5, "#444455"), R(x + 9, GY - 5 - hh, 16, hh, "#cfd3e6"),
                      R(x + 9, GY - 5 - hh, 5, hh, "#ffffff"), R(x, GY - 10 - hh, 34, 5, "#ff4d6d"),
                      R(x + 2, GY - 10 - hh, 14, 2, "#ff9ab0")]
            elif k == "bot":
                col = "#ff4d6d" if not ob["dead"] else "#7a2a38"
                l = (t >> 3) & 1
                s += [R(x - 1, y + 7, 36, 24, "#4a0f1f"), R(x, y + 8, 34, 22, col), R(x, y + 22, 34, 8, "#c9304f"),
                      R(x + 3, y - 1, 28, 12, "#4a0f1f"), R(x + 4, y, 26, 10, "#d94566"),
                      R(x + 6, y + 12, 9, 9, "#ffffff"), R(x + 20, y + 12, 9, 9, "#ffffff"),
                      R(x + 9, y + 15, 4, 5, "#111111"), R(x + 23, y + 15, 4, 5, "#111111"),
                      R(x + 4 + l * 2, y + 30, 9, 4, "#444444"), R(x + 21 - l * 2, y + 30, 9, 4, "#444444"),
                      R(x + 15, y - 6, 4, 8, "#999999"), C(x + 17, y - 9, 4, "#ffd23f")]
                if hq and not ob["dead"]:
                    s.append(C(x + 17, y - 9, 9, "#33ffd23f"))

        if g.boss is not None:
            draw_boss(s, g.boss, ox)

        draw_player(s, ox)

        for p in g.parts:
            if p.get("k") == "d":
                if hq:
                    s.append(C(p["x"], p["y"], 2 + (16 - p["l"]) * 0.35, p["c"]))
            else:
                s.append(R(p["x"], p["y"], 5, 5, p["c"]))
        for f in g.floaters:
            s += TS(f["t"], f["x"], f["y"], 16, "#ffd23f", True)

        # ----- HUD -----
        s += [R(8, 6, 232, 64, "#99000000"), R(8, 6, 232, 3, "#66ffffff")]
        s += TS(f"SCORE {g.score:06d}", 16, 12, 20, "#ffffff")
        s += TS(f"HI {g.hi:06d}", 16, 40, 14, "#ffd23f")
        for i in range(g.lives):
            lx = W - 44 - i * 32
            s += [R(lx - 1, 15, 24, 24, "#0a1a5a"), R(lx, 16, 22, 22, "#2b6bff"), R(lx + 9, 22, 10, 10, "#ffffff"),
                  R(lx + 14, 24, 3, 6, "#111111"), R(lx + 8, 28, 12, 7, "#f2c9a0")]
        s.append(O(W - 108, 50, 14, 20, "#ffd23f", 3))
        s += TS(f"x{g.ring_count}", W - 86, 52, 18, "#ffd23f")
        if g.boss is not None and g.boss["phase"] == "enter" and (t >> 3) & 1:
            s += [R(0, 0, W, 5, "#ffff4d6d"), R(0, H - 5, W, 5, "#ffff4d6d")]
            s += TS("WARNING!", W / 2, 70, 34, "#ff4d6d", True)
        if g.banner > 0 and (g.banner > 90 or (g.banner >> 2) & 1):
            s += TS(f"ZONE {g.boss_level + 1}", W / 2, 150, 44, "#ffffff", True)

        if g.state == "start":
            s += [R(0, 0, W, H, "#cc050820")]
            s += TS("BLITZ RUN", W / 2, 90, 62, "#ffd23f", True, "#ff4d6d")
            s += TS("TAP TO START", W / 2, 190, 28, "#ffffff", True)
            s += TS("RINGS  *  SPRINGS  *  RAMPS  *  BOSS", W / 2, 245, 16, "#9aa3e8", True)
            if (t >> 5) & 1:
                s += TS("INSERT COIN", W / 2, 300, 22, "#ff4d6d", True)
        elif g.state == "over":
            s += [R(0, 0, W, H, "#cc050820")]
            s += TS("GAME OVER", W / 2, 90, 56, "#ff4d6d", True)
            s += TS(f"SCORE {g.score}", W / 2, 180, 30, "#ffffff", True)
            s += TS("NEW HIGH SCORE!" if g.new_hi else f"HI {g.hi}", W / 2, 230, 24, "#ffd23f", True)
            s += TS(f"RINGS {g.ring_count}", W / 2, 270, 18, "#9aa3e8", True)
            if (t >> 5) & 1:
                s += TS("TAP TO RETRY", W / 2, 320, 24, "#ffffff", True)
        canvas.shapes = s

    # ---------- saved high score ----------
    async def load_hi():
        try:
            v = page.client_storage.get("blitz_hi")
            if inspect.isawaitable(v):
                v = await v
            return int(v or 0)
        except Exception:
            pass
        try:
            v = ft.SharedPreferences().get("blitz_hi")
            if inspect.isawaitable(v):
                v = await v
            return int(v or 0)
        except Exception:
            return 0

    async def save_hi(value):
        try:
            v = page.client_storage.set("blitz_hi", value)
            if inspect.isawaitable(v):
                await v
            return
        except Exception:
            pass
        try:
            v = ft.SharedPreferences().set("blitz_hi", value)
            if inspect.isawaitable(v):
                await v
        except Exception:
            pass

    # ---------- input ----------
    def on_down(e):
        g.jump()

    def on_up(e):
        g.release()

    def on_key(e: ft.KeyboardEvent):
        if e.key in (" ", "Arrow Up", "Enter"):
            g.jump()

    page.on_keyboard_event = on_key

    center = ft.Alignment(0, 0) if hasattr(ft, "Alignment") else ft.alignment.center

    def border_all(w, c):
        try:
            return ft.border.all(w, c)
        except Exception:
            return ft.Border.all(w, c)

    def shadow(dy, color, blur=0, spread=0):
        return ft.BoxShadow(spread_radius=spread, blur_radius=blur, color=color, offset=ft.Offset(0, dy))

    def side(x):
        if hasattr(ft, "Alignment"):
            return ft.Alignment(x, 0)
        return ft.alignment.center_left if x < 0 else ft.alignment.center_right

    BOLD = ft.FontWeight.BOLD
    anim = lambda ms: ft.Animation(ms, "easeOut")

    JUMP_UP = [shadow(7, "#8a6a00"), shadow(0, "#66ffd23f", 20, 2)]
    JUMP_DOWN = [shadow(2, "#8a6a00"), shadow(0, "#66ffd23f", 8, 1)]

    try:
        # ---------- arcade-style JUMP button (3D, glows, squashes when pressed) ----------
        btn = ft.Container(
            content=ft.Row(
                [ft.Text("\u25B2", size=22, weight=BOLD, color="#7a4a00"),
                 ft.Text("JUMP", size=26, weight=BOLD, color="#3a2400")],
                alignment=ft.MainAxisAlignment.CENTER, spacing=12),
            alignment=center, height=68, width=W * K, border_radius=16,
            gradient=ft.LinearGradient(begin=ft.Alignment(0, -1), end=ft.Alignment(0, 1),
                                       colors=["#fff59a", "#ffd23f", "#ffa800"]),
            border=border_all(3, "#fff8c4"), shadow=JUMP_UP,
            scale=1, animate_scale=anim(90), animate=anim(90),
        )

        # ---------- graphics switch: pill with FX badge and sliding knob ----------
        knob = ft.Container(width=18, height=18, border_radius=9, bgcolor="#ffffff")
        track = ft.Container(content=knob, width=48, height=26, border_radius=13, padding=4,
                             alignment=side(1), bgcolor="#3ddc84", animate=anim(200))
        badge = ft.Container(content=ft.Text("FX", size=12, weight=BOLD, color="#0b1030"),
                             bgcolor="#3ddc84", border_radius=8, padding=6, animate=anim(200))
        fx_text = ft.Text("GRAPHICS  HIGH", size=14, weight=BOLD, color="#e8ecff")
        chip = ft.Container(
            content=ft.Row([ft.Row([badge, fx_text], spacing=10), track],
                           alignment=ft.MainAxisAlignment.SPACE_BETWEEN,
                           vertical_alignment=ft.CrossAxisAlignment.CENTER),
            width=W * K, padding=10, border_radius=26, bgcolor="#151b4d",
            border=border_all(2, "#3ddc84"), shadow=[shadow(0, "#443ddc84", 14, 1)],
            animate=anim(200), on_click=lambda e: toggle_fx(e),
        )
        fancy = True
    except Exception:
        btn = ft.Container(
            content=ft.Text("JUMP", size=22, weight=BOLD, color="#1a1a3a"),
            alignment=center, bgcolor="#ffd23f", border_radius=10, height=64, width=W * K)
        fx_text = ft.Text("GRAPHICS: HIGH (tap to change)", size=12, color="#9aa3e8")
        chip = ft.Container(content=fx_text, padding=8, on_click=lambda e: toggle_fx(e))
        fancy = False

    def toggle_fx(e):
        g.hq = not g.hq
        on = g.hq
        try:
            if fancy:
                fx_text.value = "GRAPHICS  HIGH" if on else "GRAPHICS  LITE"
                col = "#3ddc84" if on else "#6a6f9a"
                track.bgcolor = col
                track.alignment = side(1 if on else -1)
                badge.bgcolor = col
                chip.border = border_all(2, col)
                chip.shadow = [shadow(0, "#443ddc84", 14, 1)] if on else []
                chip.update()
            else:
                fx_text.value = "GRAPHICS: HIGH (tap to change)" if on else "GRAPHICS: LITE (tap to change)"
                fx_text.update()
        except Exception:
            try:
                page.update()
            except Exception:
                pass

    def press_fx(down):
        if not fancy:
            return
        try:
            btn.scale = 0.96 if down else 1
            btn.shadow = JUMP_DOWN if down else JUMP_UP
            btn.update()
        except Exception:
            pass

    def on_btn_down(e):
        g.jump()
        press_fx(True)

    def on_btn_up(e):
        g.release()
        press_fx(False)

    page.add(
        ft.GestureDetector(content=canvas, on_tap_down=on_down, on_tap_up=on_up),
        ft.GestureDetector(content=btn, on_tap_down=on_btn_down, on_tap_up=on_btn_up),
        chip,
        ft.Text("Tap to jump (hold = higher, double jump OK). Springs = big jump, ramps = launch, stomp the boss!",
                size=12, color="#9aa3e8"),
    )

    async def loop():
        g.hi = await load_hi()
        while True:
            g.update()
            g.update()  # 30 fps render, 60 Hz game logic
            draw()
            canvas.update()
            if g.save_pending:
                g.save_pending = False
                await save_hi(g.hi)
            await asyncio.sleep(1 / 30)

    return loop()


async def amain(page: ft.Page):
    await main(page)


if __name__ == "__main__":
    if hasattr(ft, "run"):
        ft.run(amain)
    else:
        ft.app(target=amain)
