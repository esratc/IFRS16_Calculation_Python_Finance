# IFRS16_Calculation_Python-Finance-

def IFRS(m):
    import pandas as pd
    import numpy as np
    df = pd.read_excel (r'C:\deneme.xlsx')
    df1 = pd.read_excel (r'C:\IFRS16_input_file.xlsx')
    month_end = pd.date_range(df1['Contract Start Date'][m],df1['Contract End Date'][m],  freq=pd.offsets.MonthEnd(1))
    month_start = pd.date_range(df1['Contract Start Date'][m],df1['Contract End Date'][m],  freq=pd.offsets.MonthBegin(1))
    month_middle = pd.date_range(df1['Contract Start Date'][m],df1['Contract End Date'][m],  freq=pd.offsets.MonthBegin(1)) +pd.DateOffset(days=(df1['Payment Date'][m] - df1['Contract Start Date'][m]).total_seconds()/86400)
    
    month_end = month_end.to_frame(index = False)
    month_start= month_start.to_frame(index = False)
    month_middle= month_middle.to_frame(index = False)
    
    aa = pd.concat([month_middle,month_start], axis=0, ignore_index=True)
    aa.columns = ['Time_table1']
    aa.sort_values(by='Time_table1', inplace=True)

    ab = pd.concat([month_end,month_middle], axis=0, ignore_index=True)
    ab.columns = ['Time_table2']
    ab.sort_values(by='Time_table2', inplace=True)
    all_time = pd.concat([aa, ab], axis=1)
    all_time['Start_Date'] = df1['Contract Start Date'][m]
    
    all_time['Time_diff'] = all_time['Time_table2']- all_time['Time_table1']
    all_time['Time_diff']= all_time.Time_diff.apply(lambda x: x.days)
    all_time['Time_diff_küm'] = all_time['Time_table1']- all_time['Start_Date']
    all_time['Time_diff_küm']= all_time.Time_diff_küm.apply(lambda x: x.days)
    all_time = all_time.reset_index(drop=True)
    
    file_name =  r'C:\IFRS16_input_file2.xlsx'
    if m <127:
        sheet = m
        df_payment = pd.read_excel(io=file_name, sheet_name=sheet, usecols=range(0,1))
    elif  126< m < 252:
        sheet = m -126
        df_payment = pd.read_excel(io=file_name, sheet_name=sheet, usecols=range(1,2))
    elif 251< m < 378:
        sheet = m -252
        df_payment = pd.read_excel(io=file_name, sheet_name=sheet, usecols=range(2,3))
    elif 377 < m < 504:
        sheet = m -377
        df_payment = pd.read_excel(io=file_name, sheet_name=sheet, usecols=range(3,4))
    
    
    all_time2 = pd.concat([all_time, df_payment], axis=1)
    all_time2['IBR'] = df1['IBR'][m]
    if df1['Payment Term'][m] == 'Monthly':
        all_time2['Effective_Rate'] = ((1+ df1['IBR'][m]/12)**12 - 1)
    elif df1['Payment Term'][m] == 'Quarterly':
        all_time2['Effective_Rate'] = ((1+ df1['IBR'][m]/4)**4 - 1)
    elif df1['Payment Term'][m] == 'Yearly':
        all_time2['Effective_Rate'] = ((1+ df1['IBR'][m]/1)**1 - 1)
    elif df1['Payment Term'][m] == 'Half Year':
        all_time2['Effective_Rate'] = ((1+ all_time2['IBR'][m]/2)**2 - 1) 
        
    def Cumulative(lists): 
        cu_list = [] 
        length = len(all_time2.Depreciation) 
        cu_list = [sum(all_time2.Depreciation[0:x:1]) for x in range(0, length+1)] 
        return cu_list[1:]

    all_time2['Discount_Rate'] = 1 / (( 1 + all_time2['Effective_Rate']) ** (all_time2['Time_diff_küm'] / 365)) 
    all_time2['NPV'] = round(all_time2.iloc[:,5]*all_time2['Discount_Rate'] , 0)
    all_time2['Emp'] = int(len(all_time2)/2) *[1,0]
    all_time2['Depreciation'] = round((sum(all_time2['NPV'])+ df1['Prepaid'][m])  / (len(all_time2.Time_table1)/2) , 0)*all_time2['Emp']

    list = []*len(all_time2.Depreciation)
    list.append(round((sum(all_time2['NPV']) + df1['Prepaid'][m] ),0))
    for i in range(1,len(all_time2.Depreciation)):
        list.append(list[i-1] - round(all_time2.Depreciation[i],0) )

    all_time2['Rou_opening'] = list
    all_time2['Rou_closing'] = all_time2['Rou_opening'] - all_time2['Depreciation']
    
    list3 = []*len(all_time2.Depreciation)
    list3.append(0)
    for i in range(0,len(all_time2.Depreciation)-1):
        list3.append( (all_time2.Time_table1[i+1] - all_time2.Time_table1[i] ) ) 

    all_time2['Timediff_LL'] = list3
    all_time2['Timediff_LL'] = all_time2['Timediff_LL'].astype('timedelta64[D]')
    all_time2['Timediff_LL'] = all_time2.Timediff_LL.apply(lambda x: x.days)
    all_time2['Timediff_LL'] = all_time2['Timediff_LL'] / 365
    
    list2 = []*len(all_time2.Depreciation)
    list2.append(round(sum(all_time2['NPV']),0))
    for i in range(1,len(all_time2.Depreciation)):
        list2.append(list2[i-1] + list2[i-1]*((1+ all_time2['Effective_Rate'][i-1]) ** all_time2['Timediff_LL'][i-1]) - list2[i-1] - all_time2.iloc[:,5][i-1])

    all_time2['Lease_Liability_Opening'] = list2
    all_time2['Lease_Liability_Opening'] = round(all_time2['Lease_Liability_Opening'],0)        
    all_time2['Interest_Accrued'] =  all_time2['Lease_Liability_Opening']*((1+ all_time2['Effective_Rate']) ** all_time2['Timediff_LL']) - all_time2['Lease_Liability_Opening']
    all_time2['Interest_Accrued'] = round(all_time2['Interest_Accrued'],0)   
    all_time2['Lease_Liability_Closing'] = all_time2['Lease_Liability_Opening'] + all_time2['Interest_Accrued'] - all_time2.iloc[:,5]


    all_time2['Interest_Paid'] = all_time2['Interest_Accrued']
    all_time2['Lease_Repayment'] = all_time2.iloc[:,5] - all_time2['Interest_Paid']
    all_time2 = all_time2.drop(all_time2.columns[[ 6, 8]], axis=1)
    all_time2['Site_Name'] = df1['Site Name'][m]
    all_time2['GFO'] = df1['GFO'][m]
    all_time2 = all_time2.astype({"Payment":'int', "NPV":'int', "Depreciation":'int' , "Rou_closing":'int' ,"Rou_opening":'int' , "Timediff_LL":'int' ,"Lease_Liability_Opening":'int' , "Interest_Accrued":'int', "Lease_Liability_Closing":'int',"Interest_Paid":'int', "Lease_Repayment":'int' }) 
    all_time2['Month'] = all_time2['Time_table1'].dt.month_name()
    all_time2['Year'] = all_time2['Time_table1'].dt.year
    return all_time2
