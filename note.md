## 1. Building the WRF Core

WRF软件包：
![image-20241105144329553](.\attachment\image-20241105144329553.png)

文件夹dyn_em: 求解**真实大气**的动力学过程

![image-20241105144438514](./attachment/image-20241105144438514-1731422497355-1.png)

```shell
./clean -a  # 清除编译后的可执行文件，保留源程序
```

![image-20241105144655123](./attachment/image-20241105144655123.png)

```shell
vi ~/.bashrc
source ~/.bashrc	# 配置运行环境
```

在WRF母文件夹内：

```shell
./configure
```

![image-20241105151439257](./attachment/image-20241105151439257.png)

serial: 串行

smpar: 共享内存(sm)并行(par)策略

dmpar: 分散内存(dm)并行(par)策略

![image-20241105151727047](./attachment/image-20241105151727047.png)

（这里选择1.basic）产生configure.wrf*

> [!CAUTION]
>
> 生成后还需要进一步调整部分参数以适配服务器

选择编译器（真实大气）

![image-20241105152434488](./attachment/image-20241105152434488.png)

运行后，会出现.exe可执行文件

```shell
ls -ls main/*.exe
```

![image-20241107155742601](./attachment/image-20241107155742601.png)

## 2.Building the WPS Code

> [!CAUTION]
>
> building WPS requires that WRF being already built!

![image-20241105153643705](./attachment/image-20241105153643705.png)

同样地，

```shell
./clean -a  # 清除编译后的可执行文件，保留源程序

./configure	# 选择17编译器，注意WRF_DIR等参数
# cp */configure.wps ./

./compile	# 编译
```

![image-20241107174854600](./attachment/image-20241107174854600.png)

geogrid.exe: 积分所用的区域，陆面信息

ungrib.exe: 观测 -> 网格数据解析

metgrid.exe: 插值

**对应Step1-3.**

测试区域网格：
```shell
vim namelist.wps	# 设置中心坐标、其他

ncl wrf_domain_plot.ncl	# 生成wps_show_dom.png
```

<img src="./attachment/wps_show_dom [3].png" alt="wps_show_dom [3]" style="zoom:25%;" />

```shell
&share
 wrf_core = 'ARW',
 max_dom = 1,	# dom层数
 start_date = '2018-08-01_00:00:00', '2018-08-01_00:00:00'
 end_date   = '2018-08-05_00:00:00', '2018-08-05_00:00:00'
 interval_seconds = 10800	# 对时间插值，需要时刻的初始场
 io_form_geogrid = 2,
/

&geogrid
 parent_id         =   1, 1		# 母区域驱动
 parent_grid_ratio =   1, 5		# 内网格和外网格的分辨率之比
 i_parent_start    =   1, 31	# 在单区域中，start from 1
 j_parent_start    =   1, 31	# 在多区域嵌套中，从左下角开始
 e_we              =  361, 101	# 东西方向的网格**边界**数，n网格->(n+1)边界
 e_sn              =  131, 101	# 南北方向的网格**边界**数
...
 dx = 1.0,
 dy = 1.0,	# 模式的分辨率（保持正方形）
 map_proj = 'lat-lon',	# 在lat-lon投影下dx、dy的单位为度，在其他（包括Lambert）投影下单位为米
 ref_lat   =  31.8,
 ref_lon   =  117.2,	# 中心经纬度的网格
 pole_lat = 90 ,
 pole_lon = 180,
 stand_lon =  117.2,	# 与ref_lon一致

```

Lambert投影：

```shell
&geogrid
 parent_id         =   1, 1
 parent_grid_ratio =   1, 5
 i_parent_start    =   1, 31
 j_parent_start    =   1, 31
 e_we              =  30, 101
 e_sn              =  30, 101
 !

 dx = 300000.0,
 dy = 300000.0,
 map_proj = 'lambert',
 ref_lat   =  31.8,
 ref_lon   =  117.2,
 truelat1 = 30.0 ,
 truelat2 = 60.0,
 stand_lon =  117.2,	# 做截面的起始维度

```

<img src="./attachment/wps_show_dom [6].png" alt="wps_show_dom [6]" style="zoom:25%;" />

```shell
&ungrib
 out_format = 'WPS',
 prefix = 'ungribdata/FNL',
/

```

```shell
&metgrid
 fg_name = 'ungribdata/FNL'
 io_form_metgrid = 2,
 opt_output_from_metgrid_path='finaldata/'
/
       
```

### 2.1 Runing the geogrid.exe

```shell
./geogrid.exe > geogrid.log &
```

![image-20241112145137034](./attachment/image-20241112145137034.png)

产生*.nc文件：

![image-20241112145236402](./attachment/image-20241112145236402.png)

```shell
ncview geo_em.d01.nc &	# 查看.nc文件
```

<img src="./attachment/image-20241112150354282.png" alt="image-20241112150354282" style="zoom:25%;" />

查看2d vars中的HGT_M变量（地形高度）：

![image-20241112150823854](./attachment/image-20241112150823854.png)

还包括LU_INDEX（陆面类型）：

![image-20241112151028729](./attachment/image-20241112151028729.png)

* 尝试不同的分辨率...

### 2.2 About ungrib.exe

```shell
ls -l Vtable	# 查看Vtable链接，会有不同机构的再分析数据
```

对inputdata/做重新链接...FNL*

```shell
./link_grib.csh inputdata/fnl_201808*
```

```shell
vim namelist.wps	# 编排第一块时间对齐...时间插值

./ungrgib.exe >ungrib.log &	# 运行产生ungribdata
```

![image-20241114160100709](./attachment/image-20241114160100709.png)

### 2.3 Horizontally interpolating meteorological data with metgrid

```shell
&metgrid
 fg_name = 'ungribdata/FNL'
 io_form_metgrid = 2,
 opt_output_from_metgrid_path='finaldata/'
/

```

```shell
 ./metgrid.exe >& metgrid.log &
```

![image-20241114161353540](./attachment/image-20241114161353540.png)

check:

![image-20241114162452820](./attachment/image-20241114162452820.png)

多区域嵌套时比例一般选取奇数倍（3 or 5）

```shell
&share
 wrf_core = 'ARW',
 max_dom = 2,
 start_date = '2018-08-01_00:00:00', '2018-08-01_00:00:00'
 end_date   = '2018-08-05_00:00:00', '2018-08-05_00:00:00'
 interval_seconds = 10800
 io_form_geogrid = 2,
/

&geogrid
 parent_id         =   1, 1
 parent_grid_ratio =   1, 5
 i_parent_start    =   1, 23
 j_parent_start    =   1, 22
 e_we              =  61, 71
 e_sn              =  61, 81

 geog_data_res = 'maxsnowalb_ncep+albedo_ncep+default', 'maxsnowalb_ncep+albedo_ncep+default',
 dx = 10000.0,
 dy = 10000.0,
 map_proj = 'lambert',
 ref_lat   =  31.8,
 ref_lon   =  117.2,
 truelat1 = 45.0 ,
 truelat2 = 45.0,
 stand_lon =  117.2,

```

重新运行metgrid（这一步与原始再分析数据的处理ungrib无关）：

```shell
./metgrid.exe >& metgrid.log &

```

![image-20241114165649410](./attachment/image-20241114165649410.png)

---

Test: 20km全美国

1. 备份namelist.wps和geo*.nc等数据，清除finaldata/
2. 修改namelist.wps

```shell
&geogrid
 parent_id         =   1, 1
 parent_grid_ratio =   1, 5
 i_parent_start    =   1, 23
 j_parent_start    =   1, 22
 e_we              =  451, 71
 e_sn              =  271, 81
!
 geog_data_res = 'maxsnowalb_ncep+albedo_ncep+default', 'maxsnowalb_ncep+albedo_ncep+default',
 dx = 10000.0,
 dy = 10000.0,
 map_proj = 'lambert',
 ref_lat   =  40.,
 ref_lon   =  -98.,
 truelat1 = 45.0 ,
 truelat2 = 45.0,
 stand_lon =  -98.,

```

3. 运行geogrid.exe

```shell
./geogrid.exe > geogrid.log &
```

产生geo_em.d01.nc文件，check:

![image-20241114171257146](./attachment/image-20241114171257146.png)

4. 运行metgrid.exe

```shell
./metgrid.exe &>metgrid.log &
```

check:

![image-20241114171848467](./attachment/image-20241114171848467.png)



## 3. WRF model

![image-20241114172531513](./attachment/image-20241114172531513.png)

```shel
cd test/em_real
```

![image-20241114173319513](./attachment/image-20241114173319513.png)

### 3.1 修改配置文件

check  path:

```shell
vim namelist.input
```

输入文件的绝对路径：

![image-20241114173511312](./attachment/image-20241114173511312.png)

控制模拟时间和(inputfiles)时间间隔：

<img src="./attachment/image-20241119141157716.png" alt="image-20241119141157716" style="zoom:50%;" />

* 下面的'history_interval'：每多长时间输出数据（unit: minutes）
* 'frames_per_outfile': 每个输出文件包括多少个时间段的数据

对于namelist.input文件中的dx, dy等参数：查找wps中的namelist？

> [!IMPORTANT]
>
> 在metdata/文件中查看nc文件：
>
> ```shell
> ncdump -h met_em.d01.2018-08-01_00\:00\:00.nc  >& head.txt
> ```

读取其中global_attribute: dx和dy的值，使namselist与之对齐

<img src="./attachment/image-20241119143228133.png" alt="image-20241119143228133" style="zoom:33%;" />

其他参数同样对齐:


<table><tr>
<td><img src="./attachment/image-20241119144603617.png" style="zoom:25%;" /></td>
<td><img src="./attachment/image-20241119144253948.png" alt="image-20241119144253948" style="zoom:25%;" /></td>
</tr></table>
物理过程：

```shell
&physics
mp_physics                          = 10,     3,     3,		# 云物理参数化方案
progn                               = 0,				   # 云凝结核（人类）-不考虑
ra_lw_physics                       = 4,     1,     1,
ra_sw_physics                       = 4,     1,     1,		# 考虑辐射效应的分辨率
radt                                = 10,    30,    30,		# 辐射间隔时间
sf_sfclay_physics                   = 1,     1,     1,
sf_surface_physics                  = 2,     5,     2,
bl_pbl_physics                      = 1,     5,     1,
bldt                                = 0,     0,     0,
topo_wind                           = 1,
cu_physics                          = 1,     1,     0,		# 对流参数化方案
cudt                                = 5,     5,     5,		# 对流参数化时间步长
isfflx                              = 1,
ifsnow                              = 1,
icloud                              = 1,
surface_input_source                = 1,
num_soil_layers                     = 10,
sf_urban_physics                    = 0,     0,     0,
mp_zero_out                         = 2,
mp_zero_out_thresh                  = 1.e-12
maxiens                             = 1,
maxens                              = 3,
maxens2                             = 3,
maxens3                             = 16,
ensdim                              = 144,
cu_rad_feedback                     = .false.,
sst_update                          = 1
sst_skin                            = 0
tmn_update                          = 1
usemonalb                           = .true.
num_land_cat                        = 21
/
bucket_mm                           = 1.e6
bucket_J                            = 1.e9

```

动力方程求解：

```shell
&dynamics
rk_ord                              = 3,		# runge-kutta 阶数
w_damping                           = 1,
diff_opt                            = 1,
km_opt                              = 4,
diff_6th_opt                        = 0,      0,      0,
diff_6th_factor                     = 0.12,   0.12,   0.12,
base_temp                           = 290.
damp_opt                            = 3,
zdamp                               = 1000.,  5000.,  5000.,
dampcoef                            = 0.2,    0.2,    0.2
khdif                               = 0,      0,      0,
kvdif                               = 0,      0,      0,
non_hydrostatic                     = .true., .true., .true.,
moist_adv_opt                       = 2,      1,      1,
scalar_adv_opt                      = 2,      1,      1,
chem_adv_opt                        = 2,      0,      0,
tke_adv_opt                         = 2,      0,      0,
time_step_sound                     = 4,      4,      4,
h_mom_adv_order                     = 5,      5,      5,
v_mom_adv_order                     = 3,      3,      3,
h_sca_adv_order                     = 5,      5,      5,
v_sca_adv_order                     = 3,      3,      3,
fft_filter_lat                      = 45.,
/
use_theta_m                         = 0,

```

边界条件设置：

```shell
&bdy_control
periodic_x                          = .true., .false.,.false.,		# 是否周期性？东->西
symmetric_xs                        = .false.,.false.,.false.,
symmetric_xe                        = .false.,.false.,.false.,
open_xs                             = .false.,.false.,.false.,
open_xe                             = .false.,.false.,.false.,
periodic_y                          = .false.,.false.,.false.,
symmetric_ys                        = .false.,.false.,.false.,
symmetric_ye                        = .false.,.false.,.false.,
open_ys                             = .false.,.false.,.false.,
open_ye                             = .false.,.false.,.false.,
specified                           = .true., .false.,.false.,
spec_bdy_width                      = 5,
spec_zone                           = 1,
relax_zone                          = 4,
nested                              = .false., .true., .true.,
polar                               = .false. ,.false.,.false.,
/

```

### 3.2 Running real.exe

```shell
./real.exe >& real.log &

```

check real.log...

产生三个文件：

wrfbdy_d01
wrfinput_d01
wrflowinp_d01

check wrfinput_d01:

```shell
ncview wrfinput_d01 &

```

### 3.3 Running wrf.exe

```shell
./wrf.exe &>wrf.log &

```

