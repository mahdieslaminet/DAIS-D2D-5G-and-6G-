import math
import random
import pandas as pd
from dataclasses import dataclass, field

# ===================== پارامترهای شبیه سازی =====================
NUM_DEVICES = 50
TIMESTEPS = 50
AREA_SIZE = 1000

BS_X = AREA_SIZE / 2
BS_Y = AREA_SIZE / 2

WIFI_RANGE = 200
LTE_RANGE = 600

BANDWIDTH_MHZ = 10
NOISE_FIGURE_DB = 9
NOISE_POWER_DBM = -174 + 10 * math.log10(BANDWIDTH_MHZ * 1e6) + NOISE_FIGURE_DB

TX_POWER_UE = 20
TX_POWER_RELAY = 23

PL0_DB = 40
PATHLOSS_EXP = 2.7

BATTERY_THRESHOLD = 0.75
SPEED_THRESHOLD = 15
PERC_RATE = 0.20

# ===================== توابع کمکی =====================
def dbm_to_mw(dbm):
    return 10 ** (dbm / 10)

def path_loss(distance):
    if distance < 1:
        distance = 1
    return PL0_DB + 10 * PATHLOSS_EXP * math.log10(distance)

def snr(tx_dbm, distance):
    rx_dbm = tx_dbm - path_loss(distance)
    return dbm_to_mw(rx_dbm) / dbm_to_mw(NOISE_POWER_DBM)

def shannon_rate(distance, tx_dbm):
    return BANDWIDTH_MHZ * math.log2(1 + snr(tx_dbm, distance))

# ===================== کلاس دستگاه =====================
@dataclass
class Device:
    id: int
    x: float
    y: float
    speed: float
    direction: float
    battery: float
    mode: str = "UE"
    clients: list = field(default_factory=list)

    def move(self):
        self.x = (self.x + math.cos(self.direction) * self.speed) % AREA_SIZE
        self.y = (self.y + math.sin(self.direction) * self.speed) % AREA_SIZE

    def distance_to(self, x, y):
        return math.hypot(self.x - x, self.y - y)

# ===================== مقداردهی اولیه =====================
random.seed(1)
devices = []

for i in range(NUM_DEVICES):
    devices.append(Device(
        id=i,
        x=random.uniform(0, AREA_SIZE),
        y=random.uniform(0, AREA_SIZE),
        speed=random.uniform(0, 5),
        direction=random.uniform(0, 2 * math.pi),
        battery=random.uniform(0.6, 1.0)
    ))

# انتخاب چند رله اولیه
relays = random.sample(devices, k=NUM_DEVICES // 10)
for r in relays:
    r.mode = "D2DSHR"

# ===================== حلقه اصلی شبیه سازی =====================
results = []

for t in range(TIMESTEPS):

    for d in devices:
        d.move()

    total_rate = 0
    total_power = 0

    for d in devices:
        d.clients.clear()
        dist_bs = d.distance_to(BS_X, BS_Y)
        rate_bs = shannon_rate(dist_bs, TX_POWER_UE)

        best_rate = rate_bs
        best_mode = "UE"

        for r in relays:
            if r.id == d.id:
                continue

            dist = d.distance_to(r.x, r.y)

            if dist <= WIFI_RANGE:
                rate_d2d = shannon_rate(dist, TX_POWER_RELAY)

                if rate_d2d > best_rate * (1 + PERC_RATE):
                    if d.speed < SPEED_THRESHOLD and d.battery > BATTERY_THRESHOLD:
                        best_rate = rate_d2d
                        best_mode = "D2DC"
                        r.clients.append(d.id)

        d.mode = best_mode
        total_rate += best_rate

        if best_mode == "UE":
            total_power += dbm_to_mw(TX_POWER_UE)
        else:
            total_power += dbm_to_mw(TX_POWER_RELAY)

    results.append({
        "time": t,
        "total_rate": total_rate,
        "total_power_mw": total_power
    })

# ===================== ذخیره نتایج =====================
df = pd.DataFrame(results)
df.to_csv("dais_results.csv", index=False)

print("Simulation finished successfully.")
print(df.head())
