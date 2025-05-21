import pandas as pd
import numpy_financial as npf
from datetime import datetime, timedelta

# 初始化
dates = []
cash_flows = []
operations = []
usd_amounts = []
cny_amounts = []
descriptions = []
balance = 0
u_principal = 18000
b_paths = []
rates = {2025: 0.035, 2026: 0.032, 2027: 0.03, 2028: 0.03, 2029: 0.025, 2030: 0.025, 2031: 0.025}

# 初始事件
dates.append(datetime(2025, 5, 13))
cash_flows.append(-130207.55)
operations.append("初始购汇")
usd_amounts.append("-")
cny_amounts.append(130207.55)
descriptions.append("自有 ¥45,207.55 + 贷款 ¥85,000 购汇 $18,000，汇率 7.30")

dates.append(datetime(2025, 5, 13))
cash_flows.append(8000)
operations.append("注资")
usd_amounts.append("-")
cny_amounts.append(8000.0)
descriptions.append("首月注资 ¥8,000 存入余额宝")
balance = 8000

# U 路径定存
dates.append(datetime(2025, 5, 16))
cash_flows.append(0)
operations.append("U 路径定存")
usd_amounts.append(18000.0)
cny_amounts.append("-")
descriptions.append("U 路径第 1 轮起息，$18,000，13 个月，3.50%")

# 月度循环
current_date = datetime(2025, 5, 13)
while current_date <= datetime(2030, 4, 13):
    inflow = 8000 if current_date < datetime(2028, 5, 13) else 5000
    balance += inflow
    dates.append(current_date)
    cash_flows.append(inflow)
    operations.append("注资")
    usd_amounts.append("-")
    cny_amounts.append(inflow)
    desc = f"注资 ¥{inflow:.4f}"
    if current_date in [datetime(2026, 6, 13), datetime(2026, 7, 13), datetime(2027, 5, 13), datetime(2027, 10, 13), datetime(2028, 3, 13), datetime(2028, 8, 13), datetime(2029, 4, 13), datetime(2029, 12, 13)]:
        desc += f"，余额宝增至 ¥{balance:.4f}"
    descriptions.append(desc)
    
    if current_date >= datetime(2025, 6, 13) and current_date <= datetime(2026, 5, 13):
        balance -= 7216.67
        dates.append(current_date)
        cash_flows.append(-7216.67)
        operations.append("贷款还款")
        usd_amounts.append("-")
        cny_amounts.append(7216.67)
        descriptions.append("贷款月还款 ¥7,216.67")
    
    if balance >= 36500:
        usd = balance / 7.30
        b_paths.append({'principal': usd, 'start': current_date + timedelta(days=3), 'end': current_date + timedelta(days=3) + pd.offsets.MonthBegin(13), 'rate': rates[(current_date + timedelta(days=3)).year]})
        dates.append(current_date)
        cash_flows.append(-balance)
        operations.append(f"B{len(b_paths)} 路径换汇")
        usd_amounts.append(usd)
        cny_amounts.append(balance)
        descriptions.append(f"余额宝 ¥{balance:.4f} 换汇 ${usd:.8f}")
        
        dates.append(current_date + timedelta(days=3))
        cash_flows.append(0)
        operations.append(f"B{len(b_paths)} 路径定存")
        usd_amounts.append(usd)
        cny_amounts.append("-")
        descriptions.append(f"B{len(b_paths)} 路径第 1 轮起息，${usd:.8f}，13 个月，{rates[(current_date + timedelta(days=3)).year]*100:.2f}%")
        balance = 0
    
    current_date += pd.offsets.MonthBegin(1)

# U 路径滚续
u_rollovers = [{'principal': 18000, 'start': datetime(2025, 5, 16), 'end': datetime(2026, 6, 16), 'rate': 0.035}]
current_u = u_rollovers[0]
while current_u['end'] + timedelta(days=5) <= datetime(2030, 10, 31):
    principal = current_u['principal'] * (1 + current_u['rate']) ** (13/12)
    next_start = current_u['end'] + timedelta(days=5)
    next_end = next_start + pd.offsets.MonthBegin(13)
    rate = rates[next_start.year]
    u_rollovers.append({'principal': principal, 'start': next_start, 'end': next_end, 'rate': rate})
    
    dates.append(current_u['end'])
    cash_flows.append(0)
    operations.append("U 路径定存到期滚续")
    usd_amounts.append(principal)
    cny_amounts.append("-")
    descriptions.append(f"U 路径第 {len(u_rollovers)-1} 轮到期，滚续第 {len(u_rollovers)} 轮，${principal:.8f}，13 个月，{rate*100:.2f}%，起息 {next_start.strftime('%Y-%m-%d')}")
    
    current_u = u_rollovers[-1]

# B 路径滚续
b_rollovers = []
for i, path in enumerate(b_paths, 1):
    current = path
    b_rollovers.append(current)
    round_num = 1
    while current['end'] + timedelta(days=5) <= datetime(2030, 10, 31):
        principal = current['principal'] * (1 + current['rate']) ** (13/12)
        next_start = current['end'] + timedelta(days=5)
        next_end = next_start + pd.offsets.MonthBegin(13)
        rate = rates[next_start.year]
        b_rollovers.append({'principal': principal, 'start': next_start, 'end': next_end, 'rate': rate})
        
        dates.append(current['end'])
        cash_flows.append(0)
        operations.append(f"B{i} 路径定存到期滚续")
        usd_amounts.append(principal)
        cny_amounts.append("-")
        descriptions.append(f"B{i} 路径第 {round_num} 轮到期，滚续第 {round_num+1} 轮，${principal:.8f}，13 个月，{rate*100:.2f}%，起息 {next_start.strftime('%Y-%m-%d')}")
        
        current = b_rollovers[-1]
        round_num += 1

# 额外定存
u_final = u_rollovers[-1]['principal'] * (1 + u_rollovers[-1]['rate']) ** (13/12)
u_final = u_final * (1 + 0.008) ** (12/12)
dates.append(datetime(2030, 11, 5))
cash_flows.append(0)
operations.append("U 路径定存到期额外定存")
usd_amounts.append(u_final / (1 + 0.008))
cny_amounts.append("-")
descriptions.append(f"U 路径第 5 轮到期，额外定存，${u_final / (1 + 0.008):.8f}，12 个月，0.8%，起息 2030-11-05")

dates.append(datetime(2031, 11, 5))
cash_flows.append(u_final * 7.30)
operations.append("U 路径回收")
usd_amounts.append(u_final)
cny_amounts.append(u_final * 7.30)
descriptions.append(f"U 路径额外定存到期，回收 ¥{u_final * 7.30:.4f}")

b_final = [
    {'principal': 6426.00, 'start': datetime(2030, 12, 31), 'end': datetime(2031, 12, 31), 'rate': 0.008, 'months': 12},
    {'principal': 6155.45, 'start': datetime(2031, 6, 1), 'end': datetime(2031, 12, 1), 'rate': 0.005, 'months': 6},
    {'principal': 6155.45, 'start': datetime(2031, 11, 1), 'end': datetime(2031, 12, 1), 'rate': 0.002, 'months': 1},
    {'principal': 5994.68, 'start': datetime(2031, 2, 26), 'end': datetime(2031, 8, 26), 'rate': 0.005, 'months': 6},
    {'principal': 5994.68 * (1 + 0.005) ** (6/12), 'start': datetime(2031, 8, 26), 'end': datetime(2031, 11, 26), 'rate': 0.003, 'months': 3},
    {'principal': 5994.68 * (1 + 0.005) ** (6/12) * (1 + 0.003) ** (3/12), 'start': datetime(2031, 11, 26), 'end': datetime(2031, 12, 26), 'rate': 0.002, 'months': 1},
    {'principal': 5969.09, 'start': datetime(2031, 7, 26), 'end': datetime(2031, 10, 26), 'rate': 0.003, 'months': 3},
    {'principal': 5969.09 * (1 + 0.003) ** (3/12), 'start': datetime(2031, 10, 26), 'end': datetime(2031, 11, 26), 'rate': 0.002, 'months': 1},
    {'principal': 5969.09 * (1 + 0.003) ** (3/12) * (1 + 0.002) ** (1/12), 'start': datetime(2031, 11, 26), 'end': datetime(2031, 12, 26), 'rate': 0.002, 'months': 1},
    {'principal': 5811.28, 'start': datetime(2031, 2, 21), 'end': datetime(2031, 8, 21), 'rate': 0.005, 'months': 6},
    {'principal': 5811.28 * (1 + 0.005) ** (6/12), 'start': datetime(2031, 8, 21), 'end': datetime(2031, 11, 21), 'rate': 0.003, 'months': 3},
    {'principal': 5811.28 * (1 + 0.005) ** (6/12) * (1 + 0.003) ** (3/12), 'start': datetime(2031, 11, 21), 'end': datetime(2031, 12, 21), 'rate': 0.002, 'months': 1},
    {'principal': 5783.28, 'start': datetime(2031, 10, 21), 'end': datetime(2031, 11, 21), 'rate': 0.002, 'months': 1},
    {'principal': 5783.28 * (1 + 0.002) ** (1/12), 'start': datetime(2031, 11, 21), 'end': datetime(2031, 12, 21), 'rate': 0.002, 'months': 1},
    {'principal': 5630.52, 'start': datetime(2031, 5, 16), 'end': datetime(2031, 11, 16), 'rate': 0.005, 'months': 6},
    {'principal': 5630.52 * (1 + 0.005) ** (6/12), 'start': datetime(2031, 11, 16), 'end': datetime(2031, 12, 16), 'rate': 0.002, 'months': 1}
]

b_total = 0
for i, path in enumerate(b_final, 1):
    principal = path['principal']
    rate = path['rate']
    months = path['months']
    end_principal = principal * (1 + rate) ** (months/12)
    
    if months == b_final[i-1]['months']:  # 最后一轮，回收
        dates.append(path['end'])
        cash_flows.append(end_principal * 7.30)
        operations.append(f"B{i} 路径回收")
        usd_amounts.append(end_principal)
        cny_amounts.append(end_principal * 7.30)
        descriptions.append(f"B{i} 路径额外定存到期，回收 ¥{end_principal * 7.30:.4f}")
        b_total += end_principal
    else:
        dates.append(path['start'])
        cash_flows.append(0)
        operations.append(f"B{i} 路径额外定存")
        usd_amounts.append(principal)
        cny_amounts.append("-")
        descriptions.append(f"B{i} 路径额外定存，${principal:.8f}，{months} 个月，{rate*100:.2f}%，起息 {path['start'].strftime('%Y-%m-%d')}")

# 创建 DataFrame
df = pd.DataFrame({
    'Date': dates,
    'Operation': operations,
    'USD Amount': usd_amounts,
    'CNY Amount': cny_amounts,
    'Cash Flow (CNY)': cash_flows,
    'Description': descriptions
})

# 按日期排序
df['Date'] = pd.to_datetime(df['Date'])
df = df.sort_values('Date')

# 保存为 CSV
df.to_csv('investment_plan.csv', index=False, encoding='utf-8')

# 计算指标
total_cost = 408000 + 1600.04 + 45207.55
total_recovery = u_final * 7.30 + b_total * 7.30
net_profit = total_recovery - total_cost
net_yield = net_profit / total_cost
years = (datetime(2031, 12, 1) - datetime(2025, 5, 13)).days / 365.25
annual_yield = net_yield / years
xirr = npf.irr(cash_flows) * 100 if len(cash_flows) > 1 else 0

# 追加指标到 CSV
with open('investment_plan.csv', 'a', encoding='utf-8') as f:
    f.write("\n# 最终指标：\n")
    f.write(f"# 总成本：¥{total_cost:.4f}\n")
    f.write(f"# 总回收：¥{total_recovery:.4f}\n")
    f.write(f"# 净收益：¥{net_profit:.4f}\n")
    f.write(f"# 净收益率：{net_yield*100:.4f}%\n")
    f.write(f"# 平均年化收益率：{annual_yield*100:.4f}%（{years:.2f} 年）\n")
    f.write(f"# XIRR：≈{xirr:.4f}%")

print("CSV 文件已生成：investment_plan.csv")
