# why——do——this——script？  
市面上制造msd的后处理程序真的写得太烂了，反正笔者能力有限并没有搜索到特别ok的，所以自己没事编写了这个脚本  
脚本编写逻辑也很简单  
读取数据，转化数据，带入msd公式打印结果，利用excel做储存
所需文件由vasp计算得到的**XDATCAR**
```python
import numpy as np
import pandas as pd


with open('.\XDATCAR') as f:
    temp=f.readlines()
box=[ ]
for i in range(2,5):
    print(temp[i].rstrip('\n').split())
    box.append(list(map(float,temp[i].rstrip('\n').split())))
box=np.array(box)

##--up 型box直接乘就是实际位置，--down型要乘上矩阵[[0 0 1][0 1 0][1 0 0]]
def judgeboxsttri(box):
    if box[1][0]!=0 or box[2][1]!=0 or box[2][0]!=0:return 'flag_down' ;
    else: return 'flag_up';
print(box,'flag:',judgeboxsttri(box))
for i in range(5,6):
    elementlist=temp[i].rstrip('\n').split()
for i in range(6,7):
    elementlistnum=list(map(int,temp[i].rstrip('\n').split()))
elementdict={}
elementtotalnum=0
for i in range(len(elementlistnum)):
    elementdict[elementlist[i]]=elementlistnum[i]
    elementtotalnum+=elementlistnum[i]
print(elementtotalnum)
n_frame=int((len(temp)-7)/(elementtotalnum+1))
print(n_frame)

# def lineelementflag(elementlist,elementlistnum,linestr):
#     for i in ele

elementlistorder=elementlistnum[:]
for i in range(len(elementlistnum)):
    elementlistorder[i]=0
    for ii in range(i+1):
        elementlistorder[i]+=elementlistnum[ii]
def judgeposition(a,b):
    min_=[]
    for i in range(len(b)):
        if a<=b[i]:
            min_.append(i)
    return min(min_)
print(judgeposition(-2,elementlistorder))
frames_element_positionlist = []
for i in range(7,len(temp)):
    ii=-1
    if i%(elementtotalnum+1)==7:
        ii+=1
        elementbyframe = {}
    else :
        tmp=judgeposition((((i-7)%(elementtotalnum+1))),elementlistorder)
        if not elementbyframe.__contains__(elementlist[tmp]):
            elementbyframe[elementlist[tmp]]=[]
            elementbyframe[elementlist[tmp]].append(list(map(float,temp[i].rstrip('\n').split())))
        else:
            elementbyframe[elementlist[tmp]].append(list(map(float,temp[i].rstrip('\n').split())))
        if (((i-7)%(elementtotalnum+1)))==elementtotalnum:
            frames_element_positionlist.append(elementbyframe)

print(elementlist)


#----帧之间的实际位移
def twoframesmovement(list1,list2,box,boxattr):
    interval=np.array(list2)-np.array(list1)
    for i in range(len(interval)):
        if abs(interval[i])>0.5:
            list2[i]+=1

    reverse = [[0, 0, 1], [0, 1, 0], [1, 0, 0]]
    if boxattr=='flag_up':
        aa=(np.array(list2)-np.array(list1))@np.array(box)
    else:
        aa=(np.array(list2)-np.array(list1))@np.array(box)@np.array(reverse)

    return (list(aa)[0]**2+list(aa)[1]**2+list(aa)[2]**2)**0.5

#---msd
def msd_twoframesmovement(list1,list2,box,boxattr):
    reverse = [[0, 0, 1], [0, 1, 0], [1, 0, 0]]
    if boxattr=='flag_up':
        aa=(np.array(list2)-np.array(list1))@np.array(box)
    else:
        aa=(np.array(list2)-np.array(list1))@np.array(box)@np.array(reverse)
    return list(aa)[0]**2,list(aa)[1]**2,list(aa)[2]**2,(list(aa)[0]**2+list(aa)[1]**2+list(aa)[2]**2)

#储存不同帧下的运动数据 [总帧数][元素][第几个]
print(type(frames_element_positionlist[0]['H'][0]))
print(elementdict)
print(n_frame)



select_element=input("there are elements for you to choose to compute\n"+str(elementdict)+"\nplease select one!!!\n")
select_element_np=np.ones((elementdict[select_element],int(n_frame)))
print(select_element_np.shape)
for i in range(elementdict[select_element]):
    for ii in range(n_frame):
        if ii == 0:
            select_element_np[i][0]=0
        else:
            select_element_np[i][ii]=twoframesmovement(frames_element_positionlist[ii-1][select_element][i],frames_element_positionlist[ii][select_element][i],box,judgeboxsttri(box))

select_element_dp=pd.DataFrame(select_element_np)
writer = pd.ExcelWriter('某元素所有原子帧之间飘逸距离(单位Am).xlsx')  #关键2，创建名称为hhh的excel表格
select_element_dp.to_excel(writer,'page_1',float_format='%.5f',header=None,index=False)  #关键3，float_format 控制精度，将data_df写到hhh表格的第一页中。若多个文件，可以在page_2中写入
writer.save()  #关键4

select_element_average_np=np.ones(n_frame)
for ii in range(n_frame):
    aver_tmp=0
    for i in range(elementdict[select_element]):
        aver_tmp+=select_element_np[i][ii]
    select_element_average_np[ii]=aver_tmp/elementdict[select_element]
select_element_average_sum_np=np.ones(n_frame)
for i in range(n_frame):
    tmp_average_sum=0
    if i==0:
        select_element_average_sum_np[0]=select_element_average_np[0]
    else:
        for ii in range(i+1):
            tmp_average_sum+=select_element_average_np[ii]
        select_element_average_sum_np[i]=tmp_average_sum**2

select_element_average_sum_dp=pd.DataFrame(select_element_average_sum_np)
writer = pd.ExcelWriter('某元素所有原子平均飘逸距离累计(单位Am**2).xlsx')  #关键2，创建名称为hhh的excel表格
select_element_average_sum_dp.to_excel(writer,'page_1',float_format='%.5f',header=None,index=False)  #关键3，float_format 控制精度，将data_df写到hhh表格的第一页中。若多个文件，可以在page_2中写入
writer.save()  #关键4

select_element_all_msd_np=np.ones((elementdict[select_element],n_frame,4))
select_element_average_msd_np=np.ones((n_frame,4))
for i in range(elementdict[select_element]):
    for ii in range(n_frame):
        if ii == 0:
            select_element_all_msd_np[i][0]=(0,0,0,0)
        else:
            select_element_all_msd_np[i][ii]=msd_twoframesmovement(frames_element_positionlist[0][select_element][i],frames_element_positionlist[ii][select_element][i],box,judgeboxsttri(box))
for ii in range(n_frame):
    tmp_msd=(0,0,0,0)
    for i in range(elementdict[select_element]):
        tmp_msd+=select_element_all_msd_np[i][ii]
    select_element_average_msd_np[ii]=tmp_msd/elementdict[select_element]
select_element_average_msd_pd=pd.DataFrame(select_element_average_msd_np)
writer = pd.ExcelWriter('某元素msd(单位Am**2).xlsx')  #关键2，创建名称为hhh的excel表格
select_element_average_msd_pd.to_excel(writer,'page_1',float_format='%.5f',header=None,index=False)  #关键3，float_format 控制精度，将data_df写到hhh表格的第一页中。若多个文件，可以在page_2中写入
writer.save()  #关键4
```
