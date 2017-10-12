---
# LCD点亮流程及Porting guide
---

##### 一、[lk 中 LCD 点亮流程](#1)
##### 二、[LCD Porting guide](#2)
##### 三、[LCD 兼容](#3)
##### 四、[kernel 中 panel 的点亮过程](#4)
##### 五、[DSI command](#5)
##### 六、[LCD debug](#6)

<h2 id="1">一、lk 中 LCD 点亮流程</h2>

_工作目录：_ `bootable/bootloader/lk`

LK启动在`./app/aboot/aboot.c +4116 aboot_init()`
#### 1. 在 `aboot_init()` 中调用 `target_display_init()` 初始化display

```c
/* ./app/aboot/aboot.c +4116 aboot_init() */

4146 #if ENABLE_WBC                                                                   
4157                 /* Wait if the display shutdown is in progress */                
4158                 while(pm_app_display_shutdown_in_prgs());                        
4159                 if (!pm_appsbl_display_init_done())                              
4160                         target_display_init(device.display_panel); /* target/msmxxxx/target_display.c */
4161                 else                                                             
4162                         display_image_on_screen();                               
4163 #else                                                                            
4164                 target_display_init(device.display_panel);                       
4165 #endif                                         
```

#### 2. 在target_display当中，选择要点亮的panel，然后进行gcdb_display_init
```c
/* target/msmxxxx/target_display.c +575 */
575 void target_display_init(const char *panel_name)
	···
594         do {
595                 while(!lcd_panel_select(i++)) /* 设置当前要init的panel id 为多panel做兼容 */
596                 {                                                                
597                         target_force_cont_splash_disable(false);
598                         ret = gcdb_display_init(oem.panel, MDP_REV_50, (void *)MIPI_FB_ADDR);
									/* dev/gcdb/display/gcdb_display.c */

599                         if (!ret || ret == ERR_NOT_SUPPORTED) {
600                                 break;
601                         } else {                                                 
602                                 target_force_cont_splash_disable(true);
603                                 msm_display_off();
604                         }
605                 }
606         } while (++panel_loop <= oem_panel_max_auto_detect_panels());

```
#### 3. gcdb_display_init 配置power on必须的参数

##### 3.1 在 `./target/msmxxxx/oem_panel.c` 中选择对应的panel id，并填充参数
```c
/* dev/gcdb/display/gcdb_display.c */
512 int gcdb_display_init(const char *panel_name, uint32_t rev, void *base)            
513 {                                                                                
514         int ret = NO_ERROR;                                                      
515         int pan_type;                                                            
516                                                                                  
517         dsi_video_mode_phy_db.pll_type = DSI_PLL_TYPE_28NM;                      
518         pan_type = oem_panel_select(panel_name, &panelstruct, &(panel.panel_info),
519                                  &dsi_video_mode_phy_db);
	/* ./target/msmxxxx/oem_panel.c */

   
```

- oem_panel_select 中指定 panel_id 并填充


```c

1001 int oem_panel_select(const char *panel_name, struct panel_struct *panelstruct,
1002                         struct msm_panel_info *pinfo,                         
1003                         struct mdss_dsi_phy_ctrl *phy_db)                     
1004 {
···
···
1063                 if (platform_is_msm8937()){                            
1064                         /* QRD 8937 HIPAD HP50B65 (48) uses ILI9881C */
1065                         if (plat_hw_ver_major > 47) {                  
1066                                 panel_id = ILI9881_720P_VIDEO_PANEL;   
1067                         }                                              
1068                         else if (plat_hw_ver_major > 16) {             
1069                                 panel_id = HX8394F_720P_VIDEO_PANEL;   
1070                         } else {                                       
1071                                 panel_id = HX8399C_1080P_VIDEO_PANEL;  
1072                         }                                              
1073                                                                        
1074 //                      panel_id = HX8399A_1080P_VIDEO_PANEL;          
1075 //                      panel_id = NT35532_TXD_1080P_VIDEO_PANEL;      
1076                         panel_id = target_lcd_list[current_select_id]; 

#if 0	
		 //while(!lcd_panel_select(i++)) 设置当前要init的panel id,为多panel做兼容 
		 155 static int current_select_id = 0;                                                
		 156                                                                                  
		 157 static int target_lcd_list[] = {                                                 
		 158         HX8399A_1080P_VIDEO_PANEL, NT35532_TXD_1080P_VIDEO_PANEL,                
		 159 };                                                                               
		 160                                                                                  
		 161 int lcd_panel_select(int id)                                                     
		 162 {                                                                                
		 163         int count = sizeof(target_lcd_list)/sizeof(int);                         
		 164         if (id < 0 || id >= count)                                               
		 165         {                                                                        
		 166                 dprintf(CRITICAL, "-----lcd_panel_select, invalid panel index id: %d\n", id);
		 167                 return -1;                                                       
		 168         }                                                                        
		 169         current_select_id = id; /*  current_select_id == i，设置当前init的panel id */
		 170                                                                                  
		 171         return 0;                                                                
		 172 }
#endif

1077                 }
···
···
1108 panel_init:                                                               
1109         /*                                                                
1110          * Update all data structures after 'panel_init' label. Only panel
1111          * selection is supposed to happen before that.                   
1112          */                                                               
1113         if (platform_is_msm8956())                                        
1114                 memcpy(panel_regulator_settings,                          
1115                         dcdc_regulator_settings_hpm, REGULATOR_SIZE);     
1116         else                                                              
1117                 memcpy(panel_regulator_settings,                          
1118                         dcdc_regulator_settings_lpm, REGULATOR_SIZE);     
1119         pinfo->pipe_type = MDSS_MDP_PIPE_TYPE_RGB;                        
1120         return init_panel_data(panelstruct, pinfo, phy_db);/* 填充panel数据，并返回panel接口类型 */
1121 }                                                                         


                                                            
/*init_pannel_data*/

 216 static int init_panel_data(struct panel_struct *panelstruct,          
 217                         struct msm_panel_info *pinfo,                 
 218                         struct mdss_dsi_phy_ctrl *phy_db)             
 219 {                                                                     
 220         int pan_type = PANEL_TYPE_DSI; /* return pan_type; */
 221         struct oem_panel_data *oem_data = mdss_dsi_get_oem_data_ptr();
 222                            
 223         switch (panel_id) { 
		/* 根据pannel_id 填充对应的数据
		 * panel_id = target_lcd_list[current_select_id];
		 */
···
···
/*
 * 填充数据
 */
 921         case NT35532_TXD_1080P_VIDEO_PANEL:                                      
 922                 panelstruct->paneldata    = &nt35532_txd_1080p_video_panel_data; 
 923                 panelstruct->panelres     = &nt35532_txd_1080p_video_panel_res;  
 924                 panelstruct->color        = &nt35532_txd_1080p_video_color;      
 925                 panelstruct->videopanel   = &nt35532_txd_1080p_video_video_panel;
 926                 panelstruct->commandpanel = &nt35532_txd_1080p_video_command_panel;
 927                 panelstruct->state        = &nt35532_txd_1080p_video_state;      
 928                 panelstruct->laneconfig   = &nt35532_txd_1080p_video_lane_config;
 929                 panelstruct->paneltiminginfo                                     
 930                         = &nt35532_txd_1080p_video_timing_info;                  
 931                 panelstruct->panelresetseq                                       
 932                                          = &nt35532_txd_1080p_video_reset_seq;   
 933                 panelstruct->backlightinfo = &nt35532_txd_1080p_video_backlight; 
 934                 pinfo->mipi.panel_on_cmds                                        
 935                         = nt35532_txd_1080p_video_on_command;                    
 936                 pinfo->mipi.num_of_panel_on_cmds                                 
 937                         = NT35532_TXD_1080P_VIDEO_ON_COMMAND;                    
 938                 pinfo->mipi.panel_off_cmds                                       
 939                         = nt35532_txd_1080p_video_off_command;                   
 940                 pinfo->mipi.num_of_panel_off_cmds                                
 941                         = NT35532_TXD_1080P_VIDEO_OFF_COMMAND;                   
 942                 memcpy(phy_db->timing,                                           
 943                         nt35532_txd_1080p_video_timings, TIMING_SIZE);           
 944                 pinfo->mipi.signature = NT35532_TXD_1080P_VIDEO_SIGNATURE;       
 945                 pinfo->read_id = nt35532_read_panel_id;                          
 946                 break;                                                           
···
···
 989         return pan_type; /* int pan_type = PANEL_TYPE_DSI; */
 990 }                       
```


##### 3.2 设置硬件参数以及power on必须的ops，调用 msm_display_init 正式init
```c
/* dev/gcdb/display/gcdb_display.c */                     
520                                                                                  
521         if (pan_type == PANEL_TYPE_DSI) {                                        
522                 if (update_dsi_display_config())                                 
523                         goto error_gcdb_display_init;                            
524                 target_dsi_phy_config(&dsi_video_mode_phy_db);                   
525                 mdss_dsi_check_swap_status();                                    
526                 mdss_dsi_set_pll_src();                                          
527                 if (dsi_panel_init(&(panel.panel_info), &panelstruct)) {         
528                         dprintf(CRITICAL, "DSI panel init failed!\n");           
529                         ret = ERROR;                                             
530                         goto error_gcdb_display_init;                            
531                 }                                                                
532
	/* 
	 *根据oem_panel_select
	 */                                                                                  
533                 panel.panel_info.mipi.mdss_dsi_phy_db = &dsi_video_mode_phy_db;  
534                 panel.pll_clk_func = mdss_dsi_panel_clock;                       
535                 panel.dfps_func = mdss_dsi_mipi_dfps_config;                     
536                 panel.power_func = mdss_dsi_panel_power;                         
537                 panel.pre_init_func = mdss_dsi_panel_pre_init;                   
538                 panel.bl_func = mdss_dsi_bl_enable;                              
539                 panel.dsi2HDMI_config = mdss_dsi2HDMI_config;                    
540                 /*                                                               
541                  * Reserve fb memory to store pll codes and pass                 
542                  * pll codes values to kernel.                                   
543                  */                                                              
544                 panel.panel_info.dfps.dfps_fb_base = base;                       
545                 base += DFPS_PLL_CODES_SIZE;                                     
546                 panel.fb.base = base;                                            
547                 dprintf(SPEW, "dfps base=0x%p,d, fb_base=0x%p!\n",               
548                                 panel.panel_info.dfps.dfps_fb_base, base);       
549                                                                                  
550                 panel.fb.width =  panel.panel_info.xres;                         
551                 panel.fb.height =  panel.panel_info.yres;                        
552                 panel.fb.stride =  panel.panel_info.xres;                        
553                 panel.fb.bpp =  panel.panel_info.bpp;                            
554                 panel.fb.format = panel.panel_info.mipi.dst_format;              
···
···
568         panel.fb.base = base;          
569         panel.mdp_rev = rev;           
570                                        
571         ret = msm_display_init(&panel); /*  */
572                                        
573 error_gcdb_display_init:               
574         display_enable = ret ? 0 : 1;  
575         return ret;                    
576 }                                      
```

#### 4. msm_display_init 正式开始点亮panel


```c
// ./platform/msm_shared/display.c
275 int msm_display_init(struct msm_fb_panel_data *pdata)
276 {                                                                  
277         int ret = NO_ERROR;                                        
278                                                                    
279         panel = pdata;                                             
280         if (!panel) {                                              
281                 ret = ERR_INVALID_ARGS;                            
282                 goto msm_display_init_out;                         
283         }                                                          
284                                                                    
285         /* Turn on panel */                                        
286         if (pdata->power_func)                                     
287                 ret = pdata->power_func(1, &(panel->panel_info));
					/* panel.power_func = mdss_dsi_panel_power; reset panel */


#if 0
	// ./dev/gcdb/display/gcdb_display.c
	 96 static int mdss_dsi_panel_power(uint8_t enable,                          
	 97                                 struct msm_panel_info *pinfo)            
	 98 {                                                                        
	 99         int ret = NO_ERROR;                                              
	100                                                                          
	101         if (enable) {                                                    
	102                 ret = target_ldo_ctrl(enable, pinfo);

		//target/msmxxxx/target_display.c
		/*
		535 int target_ldo_ctrl(uint8_t enable, struct msm_panel_info *pinfo)             
		536 {                                                                             
		537         int rc = 0;                                                           
		538         uint32_t ldo_num = REG_LDO6 | REG_LDO17;                              
		539                                                                               
		540         if (platform_is_msm8956())                                            
		541                 ldo_num |= REG_LDO1;                                          
		542         else                                                                  
		543                 ldo_num |= REG_LDO2;                                          
		544                                                                               
		545         if (enable) {                                                         
		546                 regulator_enable(ldo_num);                                    
		547                 mdelay(10);                                                   
		548                 rc = wled_init(pinfo);                                        
		549                 if (rc) {                                                     
		550                         dprintf(CRITICAL, "%s: wled init failed\n", __func__);
		551                         return rc;                                            
		552                 }                                                             
		553                 rc = qpnp_ibb_enable(true); /*5V boost*/                      
		554                 if (rc) {                                                     
		555                         dprintf(CRITICAL, "%s: qpnp_ibb failed\n", __func__);
		556                         return rc;                                            
		557                 }                                                             
		558                 mdelay(50);                                                   
		559         } else {                                                              
		560                 /*                                                            
		561                  * LDO1, LDO2 and LDO6 are shared with other subsystems.      
		562                  * Do not disable them.                                       
		563                  */                                                           
		564                 regulator_disable(REG_LDO17);                                 
		565         }                                                                     
		566                                                                               
		567         return NO_ERROR;                                                      
		568 }                                                                             
		
		*/

	103                 if (ret) {                                               
	104                         dprintf(CRITICAL, "LDO control enable failed\n");
	105                         return ret;                                      
	106                 }                                                        
	107                                                                          
	108                 /* Panel Reset */                                        
	109                 if (!panelstruct.paneldata->panel_lp11_init) {           
	110                         ret = mdss_dsi_panel_reset(enable);
		
		// ./dev/gcdb/display/gcdb_display.c
		/*
		 73 static uint32_t mdss_dsi_panel_reset(uint8_t enable)                       
		 74 {                                                                          
		 75         uint32_t ret = NO_ERROR;                                           
		 76         if (panelstruct.panelresetseq)                                     
		 77                 ret = target_panel_reset(enable, panelstruct.panelresetseq,
		 78                                                         &panel.panel_info);
		 79         return ret;                                                        
		 80 }
		*/

	111                         if (ret) {                                       
	112                                 dprintf(CRITICAL, "panel reset failed\n")
	113                                 return ret;                              
	114                         }                                                
	115                 }                                                        
	116                 dprintf(SPEW, "Panel power on done\n");                  
	117         } else {                                                         
	···
	···
	133         return ret;
	134 }                  			
#endif

288                                                                    
289         if (ret)                                                   
290                 goto msm_display_init_out;                         
291                                                                    
292         if (pdata->dfps_func)                                      
293                 ret = pdata->dfps_func(&(panel->panel_info));      
294                                                                    
295         /* Enable clock */                                         
296         if (pdata->clk_func)                                       
297                 ret = pdata->clk_func(1, &(panel->panel_info));    
298                                                                    
299         if (ret)                                                   
300                 goto msm_display_init_out;                         
301                                                                    
302         /* Read specifications from panel if available.            
303          * If further clocks should be enabled, they can be enabled
304          * using pll_clk_func                                      
305          */                                                        
306         if (pdata->update_panel_info)                              
307                 ret = pdata->update_panel_info();                  
308                                                                    
309         if (ret)                                                   
310                 goto msm_display_init_out;                         
311                                                                    
312         /* Enabled for auto PLL calculation or to enable           
313          * additional clocks                                       
314          */                                                        
315         if (pdata->pll_clk_func)                                   
316                 ret = pdata->pll_clk_func(1, &(panel->panel_info));
317                                                                    
318         if (ret)                                                   
319                 goto msm_display_init_out;                         
320                                                                             
321         /* pinfo prepare  */                                                
322         if (pdata->panel_info.prepare) {                                    
323                 /* this is for edp which pinfo derived from edid */         
324                 ret = pdata->panel_info.prepare();                          
325                 panel->fb.width =  panel->panel_info.xres;                  
326                 panel->fb.height =  panel->panel_info.yres;                 
327                 panel->fb.stride =  panel->panel_info.xres;                 
328                 panel->fb.bpp =  panel->panel_info.bpp;                     
329         }                                                                   
330                                                                             
331         if (ret)                                                            
332                 goto msm_display_init_out;                                  
333                                                                             
334         ret = msm_fb_alloc(&(panel->fb));                                   
335         if (ret)                                                            
336                 goto msm_display_init_out;                                  
337                                                                             
338         fbcon_setup(&(panel->fb));                                          
339         //yankun 20170314 modify for low battery charge begin               
340         //display_image_on_screen();                                        
341         //yankun 20170314 modify for low battery charge end                 
342         if ((panel->dsi2HDMI_config) && (panel->panel_info.has_bridge_chip))
343                 ret = panel->dsi2HDMI_config(&(panel->panel_info));         
344         if (ret)                                                            
345                 goto msm_display_init_out;                                  
346                                                                             
347         ret = msm_display_config(); /* send dsi cmd to set panel*/


#if 0

	// ./platform/msm_shared/display.c

	 69 int msm_display_config()                                               
	 70 {                                                                      
	 71         int ret = NO_ERROR;                                            
	 72 #ifdef DISPLAY_TYPE_MDSS                                               
	 73         int mdp_rev;                                                   
	 74 #endif                                                                 
	 75         struct msm_panel_info *pinfo;                                  
	 76                                                                        
	 77         if (!panel)                                                    
	 78                 return ERR_INVALID_ARGS;                               
	 79                                                                        
	 80         pinfo = &(panel->panel_info);                                  
	 81                                                                        
	 82         /* Set MDP revision */                                         
	 83         mdp_set_revision(panel->mdp_rev);                              
	 84                                                                        
	 85         switch (pinfo->type) {                                         
	 86 #ifdef DISPLAY_TYPE_MDSS                                               
	 87         case LVDS_PANEL:                                               
	 88                 dprintf(INFO, "Config LVDS_PANEL.\n");                 
	 89                 ret = mdp_lcdc_config(pinfo, &(panel->fb));            
	 90                 if (ret)                                               
	 91                         goto msm_display_config_out;                   
	 92                 break;                                                 
	 93         case MIPI_VIDEO_PANEL: /* dsi 模式为video模式 */                                         
	 94                 dprintf(INFO, "Config MIPI_VIDEO_PANEL.\n");           
	 95                                                                        
	 96                 mdp_rev = mdp_get_revision();                          
	 97                 if (mdp_rev == MDP_REV_50 || mdp_rev == MDP_REV_304 || 
	 98                                                 mdp_rev == MDP_REV_305)
	 99                         ret = mdss_dsi_config(panel);/* 发送ON dsi cmd */


	#if 0

		// ./platform/msm_shared/mipi_dsi.c

		634 int mdss_dsi_config(struct msm_fb_panel_data *panel)
		635 {                                                   
		636         int ret = NO_ERROR;                         
		637         struct msm_panel_info *pinfo;               
		638         struct mipi_panel_info *mipi;               
		639         struct dsc_desc *dsc = NULL;                
		640         struct mipi_dsi_cmd cmd;                    
		···
		···
		693         if (!mipi->cmds_post_tg) { 
/* 根据之前填充的信息来判断，如果没有指定pinfo->mipi.cmds_post_tg = 1; 则在这里发送dsi on cmd*/
		694                 ret = mdss_dsi_panel_initialize(mipi, mipi->broadcast);


		#if 0

			// ./platform/msm_shared/mipi_dsi.c

			 491 int mdss_dsi_panel_initialize(struct mipi_panel_info *mipi, uint32_t
			 492                 broadcast)                                              
			 493 {                                                                       
			 494         int status = 0;                                                 
			 495         uint32_t ctrl_mode = 0;                                         
			 496                                                                         
			 497 #if (DISPLAY_TYPE_MDSS == 1)                                            
			 498         if (!mipi->panel_on_cmds)                                       
			 499                 goto end;                                               
			 500                                                                         
			 501         ctrl_mode = readl(mipi->ctl_base + CTRL);                       
			 502                                                                         
			 503         /* Enable command mode before sending the commands. */          
			 504         writel(ctrl_mode | 0x04, mipi->ctl_base + CTRL);                
			 505         if (broadcast)                                                  
			 506                 writel(ctrl_mode | 0x04, mipi->sctl_base + CTRL);  

     					/* 发送从头文件中读取到的dsi cmd */
			 507         status = mdss_dsi_cmds_tx(mipi, mipi->panel_on_cmds,            
			 508                         mipi->num_of_panel_on_cmds, broadcast);         



			 509         writel(ctrl_mode, mipi->ctl_base + CTRL);                       
			 510         if (broadcast)                                                  
			 511                 writel(ctrl_mode, mipi->sctl_base + CTRL);              
			 512                                                                         
			 513         if (!broadcast && !status && target_panel_auto_detect_enabled())
			 514                 status = mdss_dsi_read_panel_signature(mipi);           
			 515                                                                         
			 516 end:                                                                    
			 517 #endif                                                                  
			 518         return status;                                                  
			 519 }                                                                       

		#endif


		695                 if (ret) {                                             
		696                         dprintf(CRITICAL, "dsi panel init error\n");   
		697                         goto error;                                    
		698                 }                                                      
		699         }                                                              

	#endif                  

	100                 else                                                   
	101                         ret = mipi_config(panel);                      
	102                                                                        
	103                 if (ret)                                               
	104                         goto msm_display_config_out;                   
	105                                                                        
	106                 if (pinfo->early_config)                               
	107                         ret = pinfo->early_config((void *)pinfo);      
	108                                                                        
	109                 ret = mdp_dsi_video_config(pinfo, &(panel->fb));       
	110                 if (ret)                                               
	111                         goto msm_display_config_out;                   
	112                 break;                                                 


#endif

                                        
348         if (ret)                                                            
349                 goto msm_display_init_out;                                  
350                                                                             
351         ret = msm_display_on(); 

#if 0

	// ./platform/msm_shared/display.c
	
	166 int msm_display_on()                                         
	167 {                                                            
	168         int ret = NO_ERROR;                                  
	169 #ifdef DISPLAY_TYPE_MDSS                                     
	170         int mdp_rev;                                         
	171 #endif                                                       
	172         struct msm_panel_info *pinfo;                        
	173                                                              
	174         if (!panel)                                          
	175                 return ERR_INVALID_ARGS;                     
	176                                                              
	177         bs_set_timestamp(BS_SPLASH_SCREEN_DISPLAY);          
	178                                                              
	179         pinfo = &(panel->panel_info);                        
	180                                                              
	181         if (pinfo->pre_on) {                                 
	182                 ret = pinfo->pre_on();                       
	183                 if (ret)                                     
	184                         goto msm_display_on_out;             
	185         }                                                    
	186                                                              
	187         switch (pinfo->type) {                               
	188 #ifdef DISPLAY_TYPE_MDSS                                     
	189         case LVDS_PANEL:                                     
	190                 dprintf(INFO, "Turn on LVDS PANEL.\n");      
	191                 ret = mdp_lcdc_on(panel);                    
	192                 if (ret)                                     
	193                         goto msm_display_on_out;             
	194                 ret = lvds_on(panel);                        
	195                 if (ret)                                     
	196                         goto msm_display_on_out;             
	197                 break;                                       
	198         case MIPI_VIDEO_PANEL:                               
	199                 dprintf(INFO, "Turn on MIPI_VIDEO_PANEL.\n");
	200                 ret = mdp_dsi_video_on(pinfo); /* 开启dsi video 模式*/             
	201                 if (ret)                                     
	202                         goto msm_display_on_out;             
	203                                                              
	204                 ret = mdss_dsi_post_on(panel); /* 发送dsi on cmd*/
	
	#if 0

	// ./platform/msm_shared/mipi_dsi.c

		 716 int mdss_dsi_post_on(struct msm_fb_panel_data *panel)                            
		 717 {                                                                                
		 718         int ret = 0;                                                             
		 719         struct msm_panel_info *pinfo = &(panel->panel_info);                     
		 720                                                                                  
		 721         if (pinfo->mipi.cmds_post_tg) {  
          /* 根据之前填充的信息来判断，如果指定了pinfo->mipi.cmds_post_tg = 1; 则在这里发送dsi on cmd*/
		 722                 ret = mdss_dsi_panel_initialize(&pinfo->mipi, pinfo->mipi.broadcast);
		  /* 与上面一样 */
		 723                 if (ret) {                                                       
		 724                         dprintf(CRITICAL, "dsi panel init error\n");             
		 725                 }                                                                
		 726         }                                                                        
		 727         return ret;                                                              
		 728 }                                                                                

	
	#endif              
	
	205                 if (ret)                                     
	206                         goto msm_display_on_out;             
	207                                                              
	208                 ret = mipi_dsi_on(pinfo);                    
	209                 if (ret)                                     
	210                         goto msm_display_on_out;             
	211                 break;                                       


#endif                                            
352         if (ret)                                                            
353                 goto msm_display_init_out;                                  
354                                                                             
358         if (pdata->post_power_func)              
359                 ret = pdata->post_power_func(1); 
360         if (ret)                                 
361                 goto msm_display_init_out;       
362                                                  
363         /* Turn on backlight */   
364         if (pdata->bl_func)                      
365                 ret = pdata->bl_func(1);/* 开启背光 */
366                                                  
367         if (ret)                                 
368                 goto msm_display_init_out;
371                                                     
372 msm_display_init_out:
373         return ret;
374 }                                                   

```


<h2 id="2">二、LCD Porting guide</h2>

---
- 参考文档：
	- 80-NU323-3SC B 多媒体驱动程序开发和调通指南 – 显示
	- 80-NN766-1 B Linux Android Display Driver Porting Guide


### _建议完整阅读这两份文档_
#### _80-NU323-3SC B 多媒体驱动程序开发和调通指南 – 显示 写的很详细，很全面_

---
## 首先调通kernel内的LCD再调试LK，如果LK配置错误，系统无法启动


#### 1. 生成头文件需要的xml，可从creatpoint获取。及放置头文件及dtsi文件
> 
	生成 .dtsi 文件和头文件
	执行下列步骤更新任何显示屏和平台的 XML 文件。
	此过程也记录在 `~/device/qcom/common/display/tools/README.txt` 中。
	1. 要使用解析器脚本，可访问 `~/device/qcom/common/display/tools/parser.pl`。
		使用 Perl 作为转换语言。用于运行脚本的命令为：
		`#perl parser.pl <”oem_panel_input_file”.xml> <panel/platform>`
	2. 使用以下命令生成面板 dtsi 和头文件：
	`#perl parser.pl panel_cmd.xml panel`
	此命令生成 `dsi-panel-cmd.dtsi` 和 `panel_cmd.h` 文件。
	3. 将 dtsi 复制到 dts 文件夹 `~/kernel/arch/arm/boot/dts/qcom`（对于 32 位版本）
		或 `~/kernel/arch/arm64/boot/dts/qcom（对于 64 位版本）`。
	4. 将头文件复制到启动加载程序 GCDB 头文件数据库 `~/bootable/bootloader/lk/dev/gcdb/display/include`。
	5. 使用以下命令为 platform-msm8610.xml 生成平台 dtsi 文件和头文件：
	`#perl parser.pl platform-msm8610.xml platform`
	此命令生成 platform_msm8610.h 和 platform-msm8610.dtsi 文件。
	6. 将 dtsi 文件的内容复制到 `~/kernel/arch/arm/boot/dts 中的 <target>-mdss.dtsi`。
	7. 将头文件的内容复制到 LK 的 `~/bootable/bootloader/lk/target/<target>/include/target/display.h`。

#### 2. kernel中的LCD配置
##### 2.1 kernel/arch/arm/boot/dts/qcom/<target>-mdss-panel.dtsi
添加 dtsi 头文件包含
`#include "dsi-panel-ili9881c-720p-video.dtsi"`
##### 2.2 kernel/arch/arm/boot/dts/qcom/<target>-pmi<xxxx>-qrd-sku7.dtsi
```dts
	&mdss_mdp {
		qcom,mdss-pref-prim-intf = "dsi";
	};

	&mdss_dsi {
		hw-config = "single_dsi";
	};

	&mdss_dsi0 {
		qcom,dsi-pref-prim-pan = <&dsi_panel_ili9881c_720p_video>; /* 指定panel */
		pinctrl-names = "mdss_default", "mdss_sleep";
		pinctrl-0 = <&mdss_dsi_active &mdss_te_active>;
		pinctrl-1 = <&mdss_dsi_suspend &mdss_te_suspend>;

		qcom,platform-te-gpio = <&tlmm 24 0>;
		qcom,platform-reset-gpio = <&tlmm 60 0>;
	};
	&mdss_dsi1 {
		status = "disabled";
	};

	/* 设置panel相关内容 */
	&dsi_panel_ili9881c_720p_video {
		qcom,panel-supply-entries = <&dsi_panel_pwr_supply>;
		  qcom,cont-splash-enabled;
	//        qcom,mdss-dsi-pan-enable-dynamic-fps;
	//        qcom,mdss-dsi-pan-fps-update = "dfps_immediate_porch_mode";
	};		
```

<h2 id="3">LCD 兼容</h2>

#### 1. LK中设置
##### 1.1 添加LCD读id的函数接口
```c
		bootable/bootloader/lk/dev/gcdb/display/include/panel_ili9881c_720p_video.h
		int ili9881c_read_panel_id(struct mipi_panel_info *mipi)
		{
		bootable/bootloader/lk/dev/gcdb/display/include/panel_nt35521_cpt_720p_video.h
		int nt35521_cpt_read_panel_id(struct mipi_panel_info *mipi)
		{
```
##### 1.2 在msm_panel_info中添加read_id函数指针
```c
		struct msm_panel_info {
		+int (*read_id) (struct mipi_panel_info *mipi);
```

##### 1.3 bootable/bootloader/lk/platform/msm_shared/mipi_dsi.c
```c
		mdss_dsi_config 
		int mdss_dsi_config(struct msm_fb_panel_data *panel)
		if (pinfo->read_id)
		{
			ret  = pinfo->read_id(mipi);
			if (ret)
				goto error;
		}
```
##### 1.4 bootable/bootloader/lk/target/msm8952/target_display.c
```c
		void target_display_init(const char *panel_name)
		+while(!lcd_pannel_select(i++))
```				
##### 1.5 bootable/bootloader/lk/target/msm8952/oem_panel.c
```c			
		static int current_select_id = 0;

		static int target_lcd_list[] = {
			IL I9881C_720P_VIDEO_PANEL, NT35521_CPT_720P_VIDEO_PANEL,
		};

		int lcd_panel_select(int id)
		{
			int count = sizeof(target_lcd_list)/sizeof(int);
			if (id < 0 || id >= count)
			{
				dprintf(CRITICAL, "-----lcd_panel_select, invalid panel index id: %d\n", id);
				return -1;
			}
			current_select_id = id;

			return 0;
		}
```

##### 1.6 `arch/arm/boot/dts/qcom/msmxxxx-pmixxxx-qrd-sku7.dtsi` 添加两个panel设置、
```c
270 &mdss_dsi0 {
271         lab-supply = <&lab_regulator>;
272         ibb-supply = <&ibb_regulator>;
273 
274         qcom,dsi-pref-prim-pan = <&dsi_panel_nt35532_txd_1080p_video>
275         pinctrl-names = "mdss_default", "mdss_sleep";
276         pinctrl-0 = <&mdss_dsi_active &mdss_te_active>;
277         pinctrl-1 = <&mdss_dsi_suspend &mdss_te_suspend>;
278 
279         qcom,platform-reset-gpio = <&tlmm 60 0>;
280 };
281 
282 &mdss_dsi1 {
283         status = "disabled";
284 };
285 
286 &labibb {
287 //      status = "disabled";
288         status = "ok";
289         qpnp,qpnp-labibb-mode = "lcd";
290 };
291 
292 &dsi_hx8399a_1080p_video {                                           
293         qcom,panel-supply-entries = <&dsi_panel_pwr_supply>;
294         qcom,cont-splash-enabled;
295 };
296 
···
···
304 &dsi_panel_nt35532_txd_1080p_video {
305         qcom,panel-supply-entries = <&dsi_panel_pwr_supply>;
306         qcom,cont-splash-enabled;
307 };

```

<h2 id="4">四、kernel 中 panel 的点亮过程</h2>

#### 1. Controller driver


##### 1.1 Register

```c

/* Controller module init*/
static int __init mdss_dsi_ctrl_driver_init(void)
{
  int ret;

  ret = mdss_dsi_ctrl_register_driver();


/**************************************************/
/* 0. call mdss_dsi_ctrl_register_driver */
/**************************************************/
  static int mdss_dsi_ctrl_register_driver(void)
  {
    return platform_driver_register(&mdss_dsi_ctrl_driver);


/**************************************************/
/* 1. call mdss_dsi_ctrl_register_driver (__platform_driver_register) */
/**************************************************/
    #define platform_driver_register(drv) \
      __platform_driver_register(drv, THIS_MODULE)
    extern int __platform_driver_register(struct platform_driver *,
              struct module *);

    int __platform_driver_register(struct platform_driver *drv,
            struct module *owner)
    {
      drv->driver.owner = owner;
      drv->driver.bus = &platform_bus_type;
      if (drv->probe)
        drv->driver.probe = platform_drv_probe;

      /* platform_drv_probe 还是会调回来 drv->probe */
      /* @platform_drv_probe */    
      ret = dev_pm_domain_attach(_dev, true);
        if (ret != -EPROBE_DEFER) {
          ret = drv->probe(dev);
          if (ret)
            dev_pm_domain_detach(_dev, true);
        }
      /* end */

      if (drv->remove)
        drv->driver.remove = platform_drv_remove;
      if (drv->shutdown)
        drv->driver.shutdown = platform_drv_shutdown;
    
      return driver_register(&drv->driver);
/**************************************************/
/* 2. call driver_register */

/**************************************************/
/**
 * driver_register - register driver with bus
 * @drv: driver to register
 *
 * We pass off most of the work to the bus_add_driver() call,
 * since most of the things we have to do deal with the bus
 * structures.
 */
      int driver_register(struct device_driver *drv)
      {
        int ret;
        struct device_driver *other;
      
        BUG_ON(!drv->bus->p);
      
        if ((drv->bus->probe && drv->probe) ||
            (drv->bus->remove && drv->remove) ||
            (drv->bus->shutdown && drv->shutdown))
          printk(KERN_WARNING "Driver '%s' needs updating - please use "
            "bus_type methods\n", drv->name);
      
        other = driver_find(drv->name, drv->bus);//查找是否驱动已经注册
        if (other) {
          printk(KERN_ERR "Error: Driver '%s' is already registered, "
            "aborting...\n", drv->name);
          return -EBUSY;
        }
      
        ret = bus_add_driver(drv);

/**************************************************/
/* 3. call bus_add_driver */
/**************************************************/
        /**
         * bus_add_driver - Add a driver to the bus.
         * @drv: driver.
         */
        int bus_add_driver(struct device_driver *drv)
        {
          struct bus_type *bus;
          struct driver_private *priv;
          int error = 0;
        
          bus = bus_get(drv->bus);
          if (!bus)
            return -EINVAL;
        
          pr_debug("bus: '%s': add driver %s\n", bus->name, drv->name);
        
          priv = kzalloc(sizeof(*priv), GFP_KERNEL);
          if (!priv) {
            error = -ENOMEM;
            goto out_put_bus;
          }
          klist_init(&priv->klist_devices, NULL, NULL);
          priv->driver = drv;
          drv->p = priv;
          priv->kobj.kset = bus->p->drivers_kset;
          error = kobject_init_and_add(&priv->kobj, &driver_ktype, NULL,
                     "%s", drv->name);
          if (error)
            goto out_unregister;
        
          klist_add_tail(&priv->knode_bus, &bus->p->klist_drivers);
          if (drv->bus->p->drivers_autoprobe) { // 默认是1
            error = driver_attach(drv);
/**************************************************/
/* 4. call driver_attach */
/**************************************************/
          /**
           * driver_attach - try to bind driver to devices.
           * @drv: driver.
           *
           * Walk the list of devices that the bus has on it and try to
           * match the driver with each one.  If driver_probe_device()
           * returns 0 and the @dev->driver is set, we've found a
           * compatible pair.
           */
          int driver_attach(struct device_driver *drv)
          {
            /**
             * bus_for_each_dev - device iterator.
             * @bus: bus type.
             * @start: device to start iterating from.
             * @data: data for the callback.
             * @fn: function to be called for each device.
             *
             * Iterate over @bus's list of devices, and call @fn for each,
             * passing it @data. If @start is not NULL, we use that device to
             * begin iterating from.
             *
             * We check the return of @fn each time. If it returns anything
             * other than 0, we break out and return that value.
             *
             * NOTE: The device that returns a non-zero value is not retained
             * in any way, nor is its refcount incremented. If the caller needs
             * to retain this data, it should do so, and increment the reference
             * count in the supplied callback.
             * 
             * int bus_for_each_dev(struct bus_type *bus, struct device *start,
                *    void *data, int (*fn)(struct device *, void *))
                *    error = fn(dev, data);
             */
            return bus_for_each_dev(drv->bus, NULL, drv, __driver_attach);
            /* drv call  __driver_attach 这个回调*/
/**************************************************/
/* 5. call __driver_attach */
/**************************************************/
            static int __driver_attach(struct device *dev, void *data)
            {
              struct device_driver *drv = data;
            
              /*
               * Lock device and try to bind to it. We drop the error
               * here and always return 0, because we need to keep trying
               * to bind to devices and some drivers will return an error
               * simply if it didn't support the device.
               *
               * driver_probe_device() will spit a warning if there
               * is an error.
               */
            
              if (!driver_match_device(drv, dev))
                return 0;
            
              if (dev->parent)  /* Needed for USB */
                device_lock(dev->parent);
              device_lock(dev);
              if (!dev->driver)
                driver_probe_device(drv, dev);

/**************************************************/
/* 6. call driver_probe_device */
/**************************************************/
              /**
               * driver_probe_device - attempt to bind device & driver together
               * @drv: driver to bind a device to
               * @dev: device to try to bind to the driver
               *
               * This function returns -ENODEV if the device is not registered,
               * 1 if the device is bound successfully and 0 otherwise.
               *
               * This function must be called with @dev lock held.  When called for a
               * USB interface, @dev->parent lock must be held as well.
               */
              int driver_probe_device(struct device_driver *drv, struct device *dev)
              {
                int ret = 0;
              
                if (!device_is_registered(dev))
                  return -ENODEV;
              
                pr_debug("bus: '%s': %s: matched device %s with driver %s\n",
                   drv->bus->name, __func__, dev_name(dev), drv->name);
              
                pm_runtime_barrier(dev);
                ret = really_probe(dev, drv);
/**************************************************/
/* 7. call really_probe */
/**************************************************/
                static int really_probe(struct device *dev, struct device_driver *drv)
                {
                  int ret = 0;
                  int local_trigger_count = atomic_read(&deferred_trigger_count);
                
                  atomic_inc(&probe_count);
                  pr_debug("bus: '%s': %s: probing driver %s with device %s\n",
                     drv->bus->name, __func__, drv->name, dev_name(dev));
                  WARN_ON(!list_empty(&dev->devres_head));
                
                  dev->driver = drv;
                
                  /* If using pinctrl, bind pins now before probing */
                  ret = pinctrl_bind_pins(dev);
                  if (ret)
                    goto probe_failed;
                
                  if (driver_sysfs_add(dev)) {
                    printk(KERN_ERR "%s: driver_sysfs_add(%s) failed\n",
                      __func__, dev_name(dev));
                    goto probe_failed;
                  }
                /*  !!!!! probe */
                  if (dev->bus->probe) {
                    ret = dev->bus->probe(dev);
                    if (ret)
                      goto probe_failed;
                  } else if (drv->probe) {
                    ret = drv->probe(dev);
                    if (ret)
                      goto probe_failed;
                  }
                
                  driver_bound(dev);
                  ret = 1;
                  pr_debug("bus: '%s': %s: bound device %s to driver %s\n",
                     drv->bus->name, __func__, dev_name(dev), drv->name);
                  goto done;
                
                probe_failed:
                  devres_release_all(dev);
                  driver_sysfs_remove(dev);
                  dev->driver = NULL;
                  dev_set_drvdata(dev, NULL);
                
                  if (ret == -EPROBE_DEFER) {
                    /* Driver requested deferred probing */
                    dev_dbg(dev, "Driver %s requests probe deferral\n", drv->name);
                    driver_deferred_probe_add(dev);
                    /* Did a trigger occur while probing? Need to re-trigger if yes */
                    if (local_trigger_count != atomic_read(&deferred_trigger_count))
                      driver_deferred_probe_trigger();
                  } else if (ret != -ENODEV && ret != -ENXIO) {
                    /* driver matched but the probe failed */
                    printk(KERN_WARNING
                           "%s: probe of %s failed with error %d\n",
                           drv->name, dev_name(dev), ret);
                  } else {
                    pr_debug("%s: probe of %s rejects match %d\n",
                           drv->name, dev_name(dev), ret);
                  }
                  /*
                   * Ignore errors returned by ->probe so that the next driver can try
                   * its luck.
                   */
                  ret = 0;
                done:
                  atomic_dec(&probe_count);
                  wake_up(&probe_waitqueue);
                  return ret;
                }
/**************************************************/
/* 7. end call really_probe */
/**************************************************/
                pm_request_idle(dev);
              
                return ret;
              }
/**************************************************/
/* 6. end call __driver_attach */
/**************************************************/
              device_unlock(dev);
              if (dev->parent)
                device_unlock(dev->parent);
            
              return 0;
            }
/**************************************************/
/* 5. end call __driver_attach */
/**************************************************/
          }
          EXPORT_SYMBOL_GPL(driver_attach);
/**************************************************/
/* 4. end call driver_attach */
/**************************************************/
            if (error)
              goto out_unregister;
          }
          module_add_driver(drv->owner, drv);
        
          error = driver_create_file(drv, &driver_attr_uevent);
          if (error) {
            printk(KERN_ERR "%s: uevent attr (%s) failed\n",
              __func__, drv->name);
          }
          error = driver_add_groups(drv, bus->drv_groups);
          if (error) {
            /* How the hell do we get out of this pickle? Give up */
            printk(KERN_ERR "%s: driver_create_groups(%s) failed\n",
              __func__, drv->name);
          }
        
          if (!drv->suppress_bind_attrs) {
            error = add_bind_files(drv);
            if (error) {
              /* Ditto */
              printk(KERN_ERR "%s: add_bind_files(%s) failed\n",
                __func__, drv->name);
            }
          }
        
          return 0;
        
        out_unregister:
          kobject_put(&priv->kobj);
          kfree(drv->p);
          drv->p = NULL;
        out_put_bus:
          bus_put(bus);
          return error;
        }

/**************************************************/
/* 3. end call bus_add_driver */
/**************************************************/

        if (ret)
          return ret;
        ret = driver_add_groups(drv, drv->groups);
        if (ret) {
          bus_remove_driver(drv);
          return ret;
        }
        kobject_uevent(&drv->p->kobj, KOBJ_ADD);
      
        return ret;
      }


/**************************************************/
/* 2. end call driver_register */
/**************************************************/

    }

/**************************************************/
/* 1. end call mdss_dsi_ctrl_register_driver (__platform_driver_register) */
/**************************************************/

  }


/**************************************************/
/* 0. end call mdss_dsi_ctrl_register_driver */
/**************************************************/

  if (ret) {
    pr_err("mdss_dsi_ctrl_register_driver() failed!\n");
    return ret;
  }

  return ret;
}
```

##### 1.2 Probe

```c
static int mdss_dsi_ctrl_probe(struct platform_device *pdev)
{
  int rc = 0;
  u32 index;
  struct mdss_dsi_ctrl_pdata *ctrl_pdata = NULL;
  struct mdss_panel_info *pinfo = NULL;
  struct device_node *dsi_pan_node = NULL;
  const char *ctrl_name;
  struct mdss_util_intf *util;

  if (!pdev || !pdev->dev.of_node) {
    pr_err("%s: pdev not found for DSI controller\n", __func__);
    return -ENODEV;
  }
  rc = of_property_read_u32(pdev->dev.of_node,
          "cell-index", &index);
  if (rc) {
    dev_err(&pdev->dev, "%s: Cell-index not specified, rc=%d\n",
      __func__, rc);
    return rc;
  }

  if (index == 0)
    pdev->id = 1;
  else
    pdev->id = 2;

  ctrl_pdata = mdss_dsi_get_ctrl(index);
  if (!ctrl_pdata) {
    pr_err("%s: Unable to get the ctrl_pdata\n", __func__);
    return -EINVAL;
  }

  platform_set_drvdata(pdev, ctrl_pdata);

  util = mdss_get_util_intf();
  if (util == NULL) {
    pr_err("Failed to get mdss utility functions\n");
    return -ENODEV;
  }

  ctrl_pdata->mdss_util = util;
  atomic_set(&ctrl_pdata->te_irq_ready, 0);

  ctrl_name = of_get_property(pdev->dev.of_node, "label", NULL);
  if (!ctrl_name)
    pr_info("%s:%d, DSI Ctrl name not specified\n",
      __func__, __LINE__);
  else
    pr_info("%s: DSI Ctrl name = %s\n",
      __func__, ctrl_name);

  rc = mdss_dsi_pinctrl_init(pdev);
  if (rc)
    pr_warn("%s: failed to get pin resources\n", __func__);

  if (index == 0) {
    ctrl_pdata->panel_data.panel_info.pdest = DISPLAY_1;
    ctrl_pdata->ndx = DSI_CTRL_0;
  } else {
    ctrl_pdata->panel_data.panel_info.pdest = DISPLAY_2;
    ctrl_pdata->ndx = DSI_CTRL_1;
  }

  if (mdss_dsi_ctrl_clock_init(pdev, ctrl_pdata)) {
    pr_err("%s: unable to initialize dsi clk manager\n", __func__);
    return -EPERM;
  }

  dsi_pan_node = mdss_dsi_config_panel(pdev, index);

/**************************************************/
/* 0. call mdss_dsi_config_panel */
/**************************************************/
  static struct device_node *mdss_dsi_config_panel(struct platform_device *pdev,
    int ndx)
  {
    struct mdss_dsi_ctrl_pdata *ctrl_pdata = platform_get_drvdata(pdev);
    char panel_cfg[MDSS_MAX_PANEL_LEN];
    struct device_node *dsi_pan_node = NULL;
    int rc = 0;
  
    if (!ctrl_pdata) {
      pr_err("%s: Unable to get the ctrl_pdata\n", __func__);
      return NULL;
    }
  
    /* DSI panels can be different between controllers */
    rc = mdss_dsi_get_panel_cfg(panel_cfg, ctrl_pdata);


/**************************************************/
/* 1.1 call mdss_dsi_get_panel_cfg */
/*
 * Form cmdline get the panel_cfg
 */
/**************************************************/
    static int mdss_dsi_get_panel_cfg(char *panel_cfg,
            struct mdss_dsi_ctrl_pdata *ctrl)
    {
      int rc;
      struct mdss_panel_cfg *pan_cfg = NULL;
    
      if (!panel_cfg)
        return MDSS_PANEL_INTF_INVALID;
    
      pan_cfg = ctrl->mdss_util->panel_intf_type(MDSS_PANEL_INTF_DSI);
      if (IS_ERR(pan_cfg)) {
        return PTR_ERR(pan_cfg);
      } else if (!pan_cfg) {
        panel_cfg[0] = 0;
        return 0;
      }
    
      pr_debug("%s:%d: cfg:[%s]\n", __func__, __LINE__,
         pan_cfg->arg_cfg);
      rc = strlcpy(panel_cfg, pan_cfg->arg_cfg,
             sizeof(pan_cfg->arg_cfg));
      return rc;
    }
/**************************************************/
/* 1.1 end call mdss_dsi_get_panel_cfg */
/**************************************************/

    if (!rc)
      /* dsi panel cfg not present */
      pr_warn("%s:%d:dsi specific cfg not present\n",
        __func__, __LINE__);
  
    /* find panel device node */
    dsi_pan_node = mdss_dsi_find_panel_of_node(pdev, panel_cfg);

/**************************************************/
/* 1.2 call mdss_dsi_find_panel_of_node */
/**************************************************/
	/**
	 * mdss_dsi_find_panel_of_node(): find device node of dsi panel
	 * @pdev: platform_device of the dsi ctrl node
	 * @panel_cfg: string containing intf specific config data
	 *
	 * Function finds the panel device node using the interface
	 * specific configuration data. This configuration data is
	 * could be derived from the result of bootloader's GCDB
	 * panel detection mechanism. If such config data doesn't
	 * exist then this panel returns the default panel configured
	 * in the device tree.
	 *
	 * returns pointer to panel node on success, NULL on error.
	 */
	static struct device_node *mdss_dsi_find_panel_of_node(
	    struct platform_device *pdev, char *panel_cfg)
	{
	  int len, i = 0;
	  int ctrl_id = pdev->id - 1;
	  char panel_name[MDSS_MAX_PANEL_LEN] = "";
	  char ctrl_id_stream[3] =  "0:";
	  char *str1 = NULL, *str2 = NULL, *override_cfg = NULL;
	  char cfg_np_name[MDSS_MAX_PANEL_LEN] = "";
	  struct device_node *dsi_pan_node = NULL, *mdss_node = NULL;
	  struct mdss_dsi_ctrl_pdata *ctrl_pdata = platform_get_drvdata(pdev);
	  struct mdss_panel_info *pinfo = &ctrl_pdata->panel_data.panel_info;
	
	  len = strlen(panel_cfg);
	  ctrl_pdata->panel_data.dsc_cfg_np_name[0] = '\0';
	  if (!len) {
	    /* no panel cfg chg, parse dt */
	    pr_debug("%s:%d: no cmd line cfg present\n",
	       __func__, __LINE__);
	    goto end;
	  } else {
	    /* check if any override parameters are set */
	    pinfo->sim_panel_mode = 0;
	    override_cfg = strnstr(panel_cfg, "#" OVERRIDE_CFG, len);
	    if (override_cfg) {
	      *override_cfg = '\0';
	      if (mdss_dsi_set_override_cfg(override_cfg + 1,
	          ctrl_pdata, panel_cfg))
	        return NULL;
	      len = strlen(panel_cfg);
	    }
	
	    if (ctrl_id == 1)
	      strlcpy(ctrl_id_stream, "1:", 3);
	
	    /* get controller number */
	    str1 = strnstr(panel_cfg, ctrl_id_stream, len);
	    if (!str1) {
	      pr_err("%s: controller %s is not present in %s\n",
	        __func__, ctrl_id_stream, panel_cfg);
	      goto end;
	    }
	    if ((str1 != panel_cfg) && (*(str1-1) != ':')) {
	      str1 += CMDLINE_DSI_CTL_NUM_STRING_LEN;
	      pr_debug("false match with config node name in \"%s\". search again in \"%s\"\n",
	        panel_cfg, str1);
	      str1 = strnstr(str1, ctrl_id_stream, len);
	      if (!str1) {
	        pr_err("%s: 2. controller %s is not present in %s\n",
	          __func__, ctrl_id_stream, str1);
	        goto end;
	      }
	    }
	    str1 += CMDLINE_DSI_CTL_NUM_STRING_LEN;
	
	    /* get panel name */
	    str2 = strnchr(str1, strlen(str1), ':');
	    if (!str2) {
	      strlcpy(panel_name, str1, MDSS_MAX_PANEL_LEN);
	    } else {
	      for (i = 0; (str1 + i) < str2; i++)
	        panel_name[i] = *(str1 + i);
	      panel_name[i] = 0;
	    }
	    pr_info("%s: cmdline:%s panel_name:%s\n",
	      __func__, panel_cfg, panel_name);
	    if (!strcmp(panel_name, NONE_PANEL))
	      goto exit;
	
	    /* get the mdss_mdp 节点，所有的panel型号挂在  mdss_mdp节点下面
	     * mdss_mdp 节点的 compatible = "qcom,mdss_mdp"  
	     */
	    mdss_node = of_parse_phandle(pdev->dev.of_node,
	      "qcom,mdss-mdp", 0);
	
	    if (!mdss_node) {
	      pr_err("%s: %d: mdss_node null\n",
	             __func__, __LINE__);
	      return NULL;
	    }
	
	    /*  根据panel_name 获取dtsi文件节点
	     *  return dsi_pan_node;
	     */
	    dsi_pan_node = of_find_node_by_name(mdss_node, panel_name);
	
	
	
	
	    if (!dsi_pan_node) {
	      pr_err("%s: invalid pan node \"%s\"\n",
	             __func__, panel_name);
	      goto end;
	    } else {
	      /* extract config node name if present */
	      str1 += i;
	      str2 = strnstr(str1, "config", strlen(str1));
	      if (str2) {
	        str1 = strnchr(str2, strlen(str2), ':');
	        if (str1) {
	          for (i = 0; ((str2 + i) < str1) &&
	               i < (MDSS_MAX_PANEL_LEN - 1); i++)
	            cfg_np_name[i] = *(str2 + i);
	          if ((i >= 0)
	            && (i < MDSS_MAX_PANEL_LEN))
	            cfg_np_name[i] = 0;
	        } else {
	          strlcpy(cfg_np_name, str2,
	            MDSS_MAX_PANEL_LEN);
	        }
	        strlcpy(ctrl_pdata->panel_data.dsc_cfg_np_name,
	          cfg_np_name, MDSS_MAX_PANEL_LEN);
	      }
	    }
	
	    /* 返回dtsi节点 */
	    return dsi_pan_node;
	  }
	end:
	  if (strcmp(panel_name, NONE_PANEL))
	    dsi_pan_node = mdss_dsi_pref_prim_panel(pdev);
	exit:
	  return dsi_pan_node;
	}
/**************************************************/
/* 1.2 end call mdss_dsi_find_panel_of_node */
/**************************************************/

    if (!dsi_pan_node) {
      pr_err("%s: can't find panel node %s\n", __func__, panel_cfg);
      of_node_put(dsi_pan_node);
      return NULL;
    }
    /* 解析panel dtsi节点中的属性，配置panel ops */
    rc = mdss_dsi_panel_init(dsi_pan_node, ctrl_pdata, ndx);

/**************************************************/
/* 1.3 call mdss_dsi_panel_init */
/**************************************************/
	int mdss_dsi_panel_init(struct device_node *node,
	  struct mdss_dsi_ctrl_pdata *ctrl_pdata,
	  int ndx)
	{
	  int rc = 0;
	  static const char *panel_name;
	  struct mdss_panel_info *pinfo;
	
	  if (!node || !ctrl_pdata) {
	    pr_err("%s: Invalid arguments\n", __func__);
	    return -ENODEV;
	  }
	
	  pinfo = &ctrl_pdata->panel_data.panel_info;
	
	  pr_debug("%s:%d\n", __func__, __LINE__);
	  pinfo->panel_name[0] = '\0';
	
	  /* 获取panel dtsi里的 mdss-dsi-panel-name */
	  panel_name = of_get_property(node, "qcom,mdss-dsi-panel-name", NULL);
	  if (!panel_name) {
	    pr_info("%s:%d, Panel name not specified\n",
	            __func__, __LINE__);
	  } else {
	    pr_info("%s: Panel Name = %s\n", __func__, panel_name);
	    strlcpy(&pinfo->panel_name[0], panel_name, MDSS_MAX_PANEL_LEN);
	  }
	
	
	  /* 解析panel节点下面其他的属性 */
	  rc = mdss_panel_parse_dt(node, ctrl_pdata);
	
	  if (rc) {
	    pr_err("%s:%d panel dt parse failed\n", __func__, __LINE__);
	    return rc;
	  }
	
	  pinfo->dynamic_switch_pending = false;
	  pinfo->is_lpm_mode = false;
	  pinfo->esd_rdy = false;
	
	  ctrl_pdata->on = mdss_dsi_panel_on; // 唤醒panel
	  ctrl_pdata->post_panel_on = mdss_dsi_post_panel_on; //唤醒屏幕的延迟操作
	  ctrl_pdata->off = mdss_dsi_panel_off; //休眠屏幕的操作
	  ctrl_pdata->low_power_config = mdss_dsi_panel_low_power_config; //针对屏幕的低功耗配置
	  ctrl_pdata->panel_data.set_backlight = mdss_dsi_panel_bl_ctrl; //panel的背光操作
	  ctrl_pdata->switch_mode = mdss_dsi_panel_switch_mode; // 模式切换
	
	  return 0;
	}
/**************************************************/
/* 1.3 end call mdss_dsi_panel_init */
/**************************************************/

    if (rc) {
      pr_err("%s: dsi panel init failed\n", __func__);
      of_node_put(dsi_pan_node);
      return NULL;
    }
  
    return dsi_pan_node;
  }
/**************************************************/
/* 0. end call mdss_dsi_config_panel */
/**************************************************/

  if (!dsi_pan_node) {
    pr_err("%s: panel configuration failed\n", __func__);
    return -EINVAL;
  }

  if (!mdss_dsi_is_hw_config_split(ctrl_pdata->shared_data) ||
    (mdss_dsi_is_hw_config_split(ctrl_pdata->shared_data) &&
    (ctrl_pdata->panel_data.panel_info.pdest == DISPLAY_1))) {
    rc = mdss_panel_parse_bl_settings(dsi_pan_node, ctrl_pdata);
    if (rc) {
      pr_warn("%s: dsi bl settings parse failed\n", __func__);
      /* Panels like AMOLED and dsi2hdmi chip
       * does not need backlight control.
       * So we should not fail probe here.
       */
      ctrl_pdata->bklt_ctrl = UNKNOWN_CTRL;
    }
  } else {
    ctrl_pdata->bklt_ctrl = UNKNOWN_CTRL;
  }

  rc = dsi_panel_device_register(pdev, dsi_pan_node, ctrl_pdata);
  if (rc) {
    pr_err("%s: dsi panel dev reg failed\n", __func__);
    goto error_pan_node;
  }

  pinfo = &(ctrl_pdata->panel_data.panel_info);
  if (!(mdss_dsi_is_hw_config_split(ctrl_pdata->shared_data) &&
    mdss_dsi_is_ctrl_clk_slave(ctrl_pdata)) &&
    pinfo->dynamic_fps) {
    rc = mdss_dsi_shadow_clk_init(pdev, ctrl_pdata);

    if (rc) {
      pr_err("%s: unable to initialize shadow ctrl clks\n",
          __func__);
      rc = -EPERM;
    }
  }

  rc = mdss_dsi_set_clk_rates(ctrl_pdata);
  if (rc) {
    pr_err("%s: Failed to set dsi clk rates\n", __func__);
    return rc;
  }

  rc = mdss_dsi_cont_splash_config(pinfo, ctrl_pdata);
  if (rc) {
    pr_err("%s: Failed to set dsi splash config\n", __func__);
    return rc;
  }

  if (mdss_dsi_is_te_based_esd(ctrl_pdata)) {
    rc = devm_request_irq(&pdev->dev,
      gpio_to_irq(ctrl_pdata->disp_te_gpio),
      hw_vsync_handler, IRQF_TRIGGER_FALLING,
      "VSYNC_GPIO", ctrl_pdata);
    if (rc) {
      pr_err("TE request_irq failed.\n");
      goto error_shadow_clk_deinit;
    }
    disable_irq(gpio_to_irq(ctrl_pdata->disp_te_gpio));
  }

  rc = mdss_dsi_get_bridge_chip_params(pinfo, ctrl_pdata, pdev);
  if (rc) {
    pr_err("%s: Failed to get bridge params\n", __func__);
    goto error_shadow_clk_deinit;
  }

  ctrl_pdata->workq = create_workqueue("mdss_dsi_dba");
  if (!ctrl_pdata->workq) {
    pr_err("%s: Error creating workqueue\n", __func__);
    rc = -EPERM;
    goto error_pan_node;
  }

  INIT_DELAYED_WORK(&ctrl_pdata->dba_work, mdss_dsi_dba_work);

  pr_info("%s: Dsi Ctrl->%d initialized, DSI rev:0x%x, PHY rev:0x%x\n",
    __func__, index, ctrl_pdata->shared_data->hw_rev,
    ctrl_pdata->shared_data->phy_rev);
  mdss_dsi_pm_qos_add_request(ctrl_pdata);

  if (index == 0)
    ctrl_pdata->shared_data->dsi0_active = true;
  else
    ctrl_pdata->shared_data->dsi1_active = true;

  /* get hipad lcd dev info*/
  strlcpy(lcd_info_pr, &pinfo->panel_name[0], MDSS_MAX_PANEL_LEN);
  hipad_device_info("lcd_info");

  return 0;

error_shadow_clk_deinit:
  mdss_dsi_shadow_clk_deinit(&pdev->dev, ctrl_pdata);
error_pan_node:
  mdss_dsi_unregister_bl_settings(ctrl_pdata);
  of_node_put(dsi_pan_node);
  return rc;
}
```

<h2 id="5">五、DSI command</h2>

#### 1. DSI command类型介绍

Hex	Bin		Description
01h	00 0001 Sync Event, V Sync Start 							Short
11h 01 0001 Sync Event, V Sync End 								Short
21h 10 0001 Sync Event, H Sync Start 							Short
31h 11 0001 Sync Event, H Sync End 								Short
08h 00 1000 End of Transmission packet (EoTp) 					Short
02h 00 0010 Color Mode (CM) Off Command 						Short
12h 01 0010 Color Mode (CM) On Command 							Short
22h 10 0010 Shut Down Peripheral Command 						Short
32h 11 0010 Turn On Peripheral Command 							Short
03h 00 0011 Generic Short WRITE, no parameters 					Short
13h 01 0011 Generic Short WRITE, 1 parameter 					Short
23h 10 0011 Generic Short WRITE, 2 parameters 					Short
04h 00 0100 Generic READ, no parameters 						Short
14h 01 0100 Generic READ, 1 parameter 							Short
24h 10 0100 Generic READ, 2 parameters 							Short
05h 00 0101 DCS Short WRITE, no parameters 						Short
15h 01 0101 DCS Short WRITE, 1 parameter 						Short
06h 00 0110 DCS READ, no parameters 							Short
37h 11 0111 Set Maximum Return Packet Size 						Short
09h 00 1001 Null Packet, no data 								Long
19h 01 1001 Blanking Packet, no data 							Long
29h 10 1001 Generic Long Write 									Long
39h 11 1001 DCS Long Write/write_LUT Command Packet 			Long
0Eh 00 1110 Packed Pixel Stream, 16-bit RGB, 5-6-5 Format 		Long
1Eh 01 1110 Packed Pixel Stream, 18-bit RGB, 6-6-6 Format 		Long
2Eh 10 1110 Loosely Packed Pixel Stream, 18-bit RGB, 6-6-6 Format Long
3Eh 11 1110 Packed Pixel Stream, 24-bit RGB, 8-8-8 Format 		Long
x0h and xFh,	xx 0000	DO NOT USE
unspecified		xx 1111	All unspecified codes are reserved

一般使用0x39(可以使用0x29替换) 和 0x05

```
带参数的
29h 10 1001 Generic Long Write
39h 11 1001 DCS Long Write/write_LUT Command Packet
```

`05h 00 0101 DCS Short WRITE, no parameters //不带参数的`

#### 2. DSI command实例

##### 2. dtsi DSI command
```c
qcom,mdss-dsi-on-command = [
		29 01 00 00 00 00 04 B9 FF 83 99
		29 01 00 00 00 00 03 C0 25 5A			
		29 01 00 00 00 00 10 B1 02 02 6D 8D 01 32 99 11 11 57 4D 56 73 02 02
		29 01 00 00 00 00 10 B2 00 88 00 AE 05 07 5A 14 00 10 00 1E 70 03 D3
		29 01 00 00 00 00 2E B4 04 FF 92 28 00 A4 00 00 0A 00 02 04 00 25 05 0C 0E 43 01 00 00 06 AF 88
							90 48 00 AA 00 00 05 00 02 04 00 2C 02 04 08 00 00 02 AF 12 00
		29 01 00 00 00 00 02 CC 00
		29 01 00 00 00 00 28 D3 10 00 01 01 00 00 30 30 32 10 04 00 04 32 10 02 00 02 00 00 00 00 00 25
							02 05 05 03 00 00 00 05 40 00 00 00 05 27 82
		29 01 00 00 00 00 21 D5 18 18 03 02 01 00 64 64 18 18 19 19 21 20 18 18 18 18 18 18 18 18 18 18
							18 18 31 31 30 30 2F 2F
		29 01 00 00 00 00 21 D6 58 58 00 01 02 03 24 24 19 19 18 18 20 21 58 58 58 58 58 58 58 58 58 58
							58 58 31 31 30 30 2F 2F
		29 01 00 00 00 00 11 D8 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
		29 01 00 00 00 00 02 BD 01
		29 01 00 00 00 00 11 D8 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
		29 01 00 00 00 00 02 BD 02
		29 01 00 00 00 00 09 D8 FF FF FF FF FF BF FF FF
		29 01 00 00 00 00 03 D3 00 14
		29 01 00 00 00 00 02 BD 00
		29 01 00 00 00 00 02 D9 84
		29 01 00 00 00 00 37 E0 01 19 24 1E 44 4C 5A 55 5D 66 6F 75 7A 81 88 8C 90 98 9A A1 95 A1 A5
				56 52 5F 6D 01 19 24 1E 44 4C 5A 55 5D 66 6F 75 7A 81 88 8C 90 98 9A A1 95 A1 A5
				56 52 5F 6D
		29 01 00 00 00 00 03 B6 85 85
		29 01 00 00 00 00 02 D2 88
		29 01 00 00 00 00 02 36 02
		29 01 00 00 78 00 02 11 00 //05 01 00 00 78 00 02 11 00
		29 01 00 00 64 00 02 29 00 
];
```

```c
/* mdss_dsi_cmd.h */
struct dsi_ctrl_hdr {
	char dtype;	/* data type */
	char last;	/* last in chain */
	char vc;	/* virtual chan */
	char ack;	/* ask ACK from peripheral */
	char wait;	/* ms */
	short dlen;	/* 16 bits */
} __packed;

struct dsi_cmd_desc {
	struct dsi_ctrl_hdr dchdr;
	char *payload;
};

/* 29 01 00 00 00 00 04 B9 FF 83 99 */

struct dsi_ctrl_hdr {
	char dtype;	/* data type */					--->0x29 //DSI command类型
	char last;	/* last in chain */				--->0x01
	char vc;	/* virtual chan */				--->0x00
	char ack;	/* ask ACK from peripheral */	--->0x00
	char wait;	/* ms */						--->0x00
	short dlen;	/* 16 bits */					--->0x00 0x04 //命令及参数的个数
} __packed;

struct dsi_cmd_desc {
	struct dsi_ctrl_hdr dchdr;
	char *payload;								--->0xB9 0xFF 0x83 0x99
};

/* B9 FF 83 99 */
B9：datasheet中命令
FF 83 99：命令的参数 


//0x05类型则没有参数 
05 01 00 00 00 00 02 11 00

```


##### 2.2 lk 头文件中的 DSI command
```c
static char hx8399c_1080p_video_on_cmd5[] = {
	0x02, 0x00, 0x39, 0xC0,
	0xCC, 0x00, 0xFF, 0xFF,
};

0x02, 0x00：命令及参数的个数
0x39：DSI command类型
0xC0：ECC校验
0xC9：datasheet中命令
0x00：命令的参数
0xFF，0xFF：数据结构补齐 

//0x05类型则没有参数 
static char hx8399c_1080p_video_on_cmd22[] = {
	0x29, 0x00, 0x05, 0x80
};
0x80：ECC校验
```

<h2 id="6">六、LCD debug</h2>

##### 1. 开启LCD ESD check

###### 1.1 修改dtsi
- 在dtsi中开启 ESD check, 添加 ESD 检测的command
> 文档：Documentation/devicetree/bindings/fb/mdss-dsi-panel.txt

```c
/* @arch/arm/boot/dts/qcom/dsi-panel-nt35532_txd_1080p_video.dtsi */
		qcom,esd-check-enabled; /* Boolean used to enable ESD recovery feature */
		qcom,mdss-dsi-panel-status-command = [06 01 00 01 05 00 01 0A];
		qcom,mdss-dsi-panel-status-command-state = "dsi_lp_mode";
		qcom,mdss-dsi-panel-status-check-mode = "reg_read";
		qcom,mdss-dsi-panel-status-read-length = <4>;
        qcom,mdss-dsi-panel-status-valid-params = <1>;
  		qcom,mdss-dsi-panel-status-value = <0x9c>;

```

###### 1.2 kenrel修改

- kernel 中开启DSI_STATUS_CHECK_DISABLE 

```c
/* @drivers/video/msm/mdss/mdss_dsi_status.c */
-#define DSI_STATUS_CHECK_DISABLE 1
+#define DSI_STATUS_CHECK_DISABLE 0

``` 



##### 2. 修改LCD clock

###### 2.1 dtsi 与 clock 相关内容

```c
/* @kernel/arch/arm/boot/dts/qcom/dsi-panel-ili9881c-hlt_720p-video.dtsi */

 25                 qcom,mdss-dsi-panel-framerate = <60>;
···
 28                 qcom,mdss-dsi-panel-width = <720>;
 29                 qcom,mdss-dsi-panel-height = <1280>;
 30                 qcom,mdss-dsi-h-front-porch = <120>;
 31                 qcom,mdss-dsi-h-back-porch = <110>;
 32                 qcom,mdss-dsi-h-pulse-width = <10>;
 33                 qcom,mdss-dsi-h-sync-skew = <0>;
 34                 qcom,mdss-dsi-v-front-porch = <20>;
 35                 qcom,mdss-dsi-v-back-porch = <18>;
 36                 qcom,mdss-dsi-v-pulse-width = <10>;
 37                 qcom,mdss-dsi-h-left-border = <0>;
 38                 qcom,mdss-dsi-h-right-border = <0>;
 39                 qcom,mdss-dsi-v-top-border = <0>;
 40                 qcom,mdss-dsi-v-bottom-border = <0>;
 41                 qcom,mdss-dsi-bpp = <24>;

```


###### 2.2 如何生成timing

参考高通文档：
- 80-NN766-1 Linux Android Display Driver Porting Guide
- 80-NH713-1 DSI Timing Parameters User Interactive Spreadsheet


a. 下载 80-NH713-1 
b. 根据 2.1 内容及平台chip填写 80-NH713-1_*.xlsm 中的 DSI and MDP registers
![](http://i.imgur.com/V3kjvsA.png)
c. 跳转到 DSI PHY timing setting，按下CTRL + J
![](http://i.imgur.com/VUWfIXR.png)

###### 2.3 细节内容

a. 计算 clock 为什么需要上面的这些参数？

- H-total = HorizontalActive + HorizontalFrontPorch + HorizontalBackPorch + HorizontalSyncPulse + HorizontalSyncSkew
- V-total = VerticalActive + VerticalFrontPorch + VerticalBackPorch + VerticalSyncPulse + VerticalSyncSkew
- Total pixel = H-total x V-total x 60 (Hz)
- Bitclk = Total pixel x bpp (byte) x 8/lane number

b. 参数的含义都是什么？
```c
/*@kernel/Documentation/devicetree/bindings/fb/mdss-dsi-panel.txt*/

116 - qcom,mdss-dsi-h-back-porch:           Horizontal back porch value in pixel.
117                                         6 = default value.
118 - qcom,mdss-dsi-h-front-porch:          Horizontal front porch value in pixel.
119                                         6 = default value.
120 - qcom,mdss-dsi-h-pulse-width:          Horizontal pulse width.
121                                         2 = default value.
122 - qcom,mdss-dsi-h-sync-skew:            Horizontal sync skew value.
123                                         0 = default value.
124 - qcom,mdss-dsi-v-back-porch:           Vertical back porch value in pixel.
125                                         6 = default value.
126 - qcom,mdss-dsi-v-front-porch:          Vertical front porch value in pixel.
127                                         6 = default value.
128 - qcom,mdss-dsi-v-pulse-width:          Vertical pulse width.
129                                         2 = default value.
130 - qcom,mdss-dsi-h-left-border:          Horizontal left border in pixel.
131                                         0 = default value
132 - qcom,mdss-dsi-h-right-border:         Horizontal right border in pixel.
133                                         0 = default value
134 - qcom,mdss-dsi-v-top-border:           Vertical top border in pixel.

```

具体的作用：
博客：
[显示技术介绍(3)_CRT技术](http://www.wowotech.net/display/crt_intro.html)
[MIPI-DSI 三种 Video Mode 理解](http://blog.csdn.net/eliot_shao/article/details/52474348)
[LCD显示的一些基本概念以及DSI的一些clock解释](http://www.cnblogs.com/biglucky/p/4142505.html)
c. LCD 如何刷屏幕的？

内容来自 MIPI_Alliance_Specification_for_Display_Serial_Interface
博客：[MIPI-DSI 三种 Video Mode 理解](http://blog.csdn.net/eliot_shao/article/details/52474348)