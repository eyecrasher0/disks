```sudo bash -c 'cat << "EOF" > /usr/local/bin/disks
#!/usr/bin/env python3
import subprocess, json, re

B, C, Y, G, R, M, E = "\033[1m", "\033[1;36m", "\033[1;33m", "\033[1;32m", "\033[1;31m", "\033[1;35m", "\033[0m"

blk = subprocess.run(["lsblk", "-d", "-o", "NAME,TYPE,RM", "-J"], capture_output=True, text=True)
raw_devs = json.loads(blk.stdout).get("blockdevices", []) if blk.stdout.strip() else []

targets = []
for d in raw_devs:
    name = d.get("name", "")
    # Берём только реальные физические диски (sdX, nvmeXnY, hdX, vdX), игнорируем zram/loop/флешки
    if d.get("type") == "disk" and not d.get("rm") and not name.startswith(("zram", "loop", "ram")):
        if name.startswith("nvme"):
            name = re.sub(r"n\d+$", "", name)
        dev_path = f"/dev/{name}"
        if dev_path not in targets:
            targets.append(dev_path)

print("\n" + "═"*72)
print(f" {B}СВОДКА ЗДОРОВЬЯ НАКОПИТЕЛЕЙ (SMART) — ДИСКОВ: {len(targets)}{E}")
print("═"*72)

for dev in targets:
    res = subprocess.run(["sudo", "smartctl", "-a", "-j", dev], capture_output=True, text=True)
    if not res.stdout.strip():
        continue
    d = json.loads(res.stdout)

    model = d.get("model_name", "Unknown")
    temp = d.get("temperature", {}).get("current", "?")
    hours = d.get("power_on_time", {}).get("hours", 0)
    years = round(hours / 8760, 1)
    cycles = d.get("power_cycle_count", "?")
    rot = d.get("rotation_rate")

    if "nvme_smart_health_information_log" in d:
        kind = f"{M}[NVMe M.2]{E}"
    elif rot == 0:
        kind = f"{C}[SATA SSD]{E}"
    elif isinstance(rot, int) and rot > 0:
        kind = f"{Y}[HDD {rot} rpm]{E}"
    else:
        kind = "[DISK]"

    print(f" {C}● {dev}{E} {kind} — {B}{model}{E}")
    print(f"   Наработка:   {hours} ч (~{years} г.) | Включений: {cycles} | Темп.: {temp}°C")

    if "nvme_smart_health_information_log" in d:
        nv = d["nvme_smart_health_information_log"]
        tbw = round((nv.get("data_units_written", 0) * 512000) / 1e12, 2)
        tbr = round((nv.get("data_units_read", 0) * 512000) / 1e12, 2)
        wear = nv.get("percentage_used", 0)
        spare = nv.get("available_spare", 100)
        err = nv.get("media_errors", 0)
        rem_y = round((years / (wear / 100)) - years, 1) if wear > 0 else "5+"
        print(f"   Запись/Чт.:  {Y}{tbw} ТБ{E} / {tbr} ТБ | Износ ячеек: {G}{wear}%{E} (ост. {100-wear}%)")
        print(f"   Здоровье:    Резерв: {spare}% | Ошибки: {err} | Прогноз: ~{rem_y} лет")

    elif "ata_smart_attributes" in d:
        attrs = {a["id"]: a for a in d["ata_smart_attributes"]["table"]}
        raw = {k: v["raw"]["value"] for k, v in attrs.items()}
        val = {k: v["value"] for k, v in attrs.items()}

        if rot == 0:
            lba_w = raw.get(241) or raw.get(246) or 0
            tbw = round((lba_w * 512) / 1e12, 2) if lba_w > 0 else "н/д"
            life = next((val[i] for i in (231, 233, 177, 202) if i in val), None)
            wear_str = f"Износ: {100 - life}% (ост. {life}%)" if life is not None else "Износ: н/д"
            bad = raw.get(5, 0) + raw.get(197, 0) + raw.get(198, 0)
            status = f"{G}Ок (0 сбоев){E}" if bad == 0 else f"{R}Сбойных блоков: {bad}{E}"
            print(f"   Запись:      {Y}{tbw} ТБ{E} | {wear_str}")
            print(f"   Здоровье:    {status}")
        else:
            bad = raw.get(5, 0) + raw.get(197, 0) + raw.get(198, 0)
            park = raw.get(193, 0)
            status = f"{G}Идеально (0 битых){E}" if bad == 0 else f"{R}Внимание! Битых: {bad}{E}"
            rem_h = max(0, 75000 - hours)
            rem_y = round(rem_h / 8760, 1)
            print(f"   Запись/Чт.:  Не отслеживается (HDD) | Парковок: {park}")
            print(f"   Здоровье:    {status} | Прогноз: ~{rem_y} лет (до 75k ч)")

    print("─"*72)
EOF
chmod +x /usr/local/bin/disks'
```

Arch linux bash script, run it, and then type
```disks```
