#!/usr/bin/env python3
"""
solar_calculator.py -- standalone CLI solar system estimator

Features:
- Estimate required system size (kW) from consumption and peak sun hours
- Estimate monthly & yearly energy production
- Recommend number of panels and area required
- Estimate installed cost, net cost after incentives
- Simple payback (years) and multi-year savings projection (with degradation & electricity escalation)
- CSV export option for yearly projection

No external dependencies. Run with: python solar_calculator.py --help
"""

from __future__ import annotations
import argparse
import math
from typing import Tuple, List, Optional

def parse_args():
    p = argparse.ArgumentParser(description="Solar system estimator (standalone, no external data).")
    group = p.add_mutually_exclusive_group(required=True)
    group.add_argument("--monthly-consumption", type=float, help="Average household consumption (kWh per month).")
    group.add_argument("--daily-consumption", type=float, help="Average household consumption (kWh per day).")

    p.add_argument("--peak-sun-hours", type=float, default=4.5,
                   help="Average peak sun hours per day (default 4.5 h/day).")
    p.add_argument("--performance-ratio", type=float, default=0.78,
                   help="System performance ratio (derate). Default 0.78 (losses, inverter, soiling).")
    p.add_argument("--panel-watt", type=float, default=400.0,
                   help="Panel rated power in Watts (default 400 W).")
    p.add_argument("--panel-area", type=float, default=1.7,
                   help="Area per panel in square meters (default 1.7 m^2 for a typical 400W panel).")
    p.add_argument("--cost-per-watt", type=float, default=2.5,
                   help="Installed system cost in $ per Watt (default $2.50/W).")
    p.add_argument("--incentive", type=float, default=0.0,
                   help="One-time incentives or rebates in $ (default 0).")
    p.add_argument("--electricity-rate", type=float, default=0.15,
                   help="Retail electricity rate in $/kWh (default $0.15).")
    p.add_argument("--electricity-escalation", type=float, default=0.02,
                   help="Annual electricity price escalation (fraction, default 0.02 = 2%).")
    p.add_argument("--degradation", type=float, default=0.005,
                   help="Annual system degradation (fraction, default 0.005 = 0.5%/yr).")
    p.add_argument("--horizon", type=int, default=25,
                   help="Analysis horizon in years (default 25).")
    p.add_argument("--round-up", action="store_true",
                   help="Force panel count to round up (default: nearest whole panel).")
    p.add_argument("--no-area", action="store_true",
                   help="Do not compute area estimate.")
    p.add_argument("--quiet", action="store_true",
                   help="Print only numeric summary (single-line).")
    p.add_argument("--csv", action="store_true",
                   help="Output savings projection in CSV format (Year,Production_kWh,Savings_USD).")
    return p.parse_args()

def consumption_from_args(monthly: Optional[float], daily: Optional[float]) -> Tuple[float, float]:
    if monthly is not None:
        monthly_kwh = monthly
        daily_kwh = monthly_kwh / 30.0
    else:
        daily_kwh = float(daily)
        monthly_kwh = daily_kwh * 30.0
    return daily_kwh, monthly_kwh

def required_system_kw(daily_kwh: float, peak_sun_hours: float, performance_ratio: float) -> float:
    if peak_sun_hours <= 0 or performance_ratio <= 0:
        raise ValueError("peak_sun_hours and performance_ratio must be > 0")
    return daily_kwh / (peak_sun_hours * performance_ratio)

def monthly_and_annual_production(system_kw: float, peak_sun_hours: float, performance_ratio: float) -> Tuple[float, float]:
    monthly = system_kw * peak_sun_hours * 30.0 * performance_ratio
    annual = monthly * 12.0
    return monthly, annual

def panels_and_area(system_kw: float, panel_watt: float, panel_area: float, round_up: bool) -> Tuple[int, float]:
    panels_exact = (system_kw * 1000.0) / panel_watt
    panels = int(math.ceil(panels_exact)) if round_up else int(round(panels_exact))
    area = panels * panel_area
    return panels, area

def cost_and_net(system_kw: float, cost_per_watt: float, incentive: float) -> Tuple[float, float]:
    base_cost = system_kw * 1000.0 * cost_per_watt
    net_cost = max(0.0, base_cost - incentive)
    return base_cost, net_cost

def simple_payback_years(net_cost: float, annual_production_kwh: float, electricity_rate: float) -> Optional[float]:
    annual_savings = annual_production_kwh * electricity_rate
    if annual_savings <= 0:
        return None
    return net_cost / annual_savings

def savings_projection(net_cost: float,
                       annual_production_kwh: float,
                       electricity_rate: float,
                       escalation: float,
                       degradation: float,
                       horizon: int) -> Tuple[float, List[float], List[float]]:
    productions: List[float] = []
    savings: List[float] = []
    prod = annual_production_kwh
    price = electricity_rate
    total = 0.0
    for _ in range(1, horizon + 1):
        productions.append(prod)
        year_saving = prod * price
        savings.append(year_saving)
        total += year_saving
        prod *= (1.0 - degradation)
        price *= (1.0 + escalation)
    return total, productions, savings

def format_currency(x: float) -> str:
    return f"${x:,.2f}"

def print_summary(args, daily_kwh, monthly_kwh, system_kw, monthly_prod, annual_prod, panels, area, base_cost, net_cost, payback, total_savings, productions, savings):
    if args.csv:
        print("Year,Production_kWh,Savings_USD")
        for i, (prod, sav) in enumerate(zip(productions, savings), start=1):
            print(f"{i},{int(prod)},{sav:.2f}")
        return

    if args.quiet:
        parts = [
            f"{system_kw:.3f} kW",
            f"{monthly_prod:.1f} kWh/mo",
            f"{annual_prod:.0f} kWh/yr",
            f"{panels} panels",
            f"base_cost={base_cost:.2f}",
            f"net_cost={net_cost:.2f}",
            f"payback_years={payback if payback is not None else 'inf'}"
        ]
        if not args.no_area:
            parts.insert(4, f"{area:.1f} m^2")
        print(", ".join(parts))
        return

    print("Solar system estimate")
    print("---------------------")
    print("Inputs:")
    print(f"  Consumption: {monthly_kwh:.1f} kWh/month ({daily_kwh:.2f} kWh/day)")
    print(f"  Peak sun hours: {args.peak_sun_hours:.2f} h/day")
    print(f"  Performance ratio: {args.performance_ratio:.3f}")
    print()
    print("Results:")
    print(f"  Required system size: {system_kw:.3f} kW ({system_kw*1000:.0f} W)")
    print(f"  Estimated production: {monthly_prod:.1f} kWh/month, {annual_prod:.0f} kWh/year")
    print(f"  Panels: {panels} x {args.panel_watt:.0f} W")
    if not args.no_area:
        print(f"  Estimated area required: {area:.1f} m^2")
    print(f"  Installed cost: {format_currency(base_cost)}")
    print(f"  Net cost after incentives: {format_currency(net_cost)}")
    print(f"  Simple payback: {f'{payback:.1f} years' if payback is not None else 'N/A'}")
    print()
    print("Savings projection:")
    print(f"  Total savings over {args.horizon} years: {format_currency(total_savings)}")
    print("  Year | Production (kWh) | Savings")
    for i, (prod, sav) in enumerate(zip(productions, savings), start=1):
        print(f"  {i:>4} | {prod:>14.0f} | {format_currency(sav)}")

def main():
    args = parse_args()
    daily_kwh, monthly_kwh = consumption_from_args(args.monthly_consumption, args.daily_consumption)
    system_kw = required_system