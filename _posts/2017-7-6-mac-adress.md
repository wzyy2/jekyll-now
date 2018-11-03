---
layout: post
title: Burn Mac Address(Rockchip Linux)
category: tips
tag: EN
comments: 1
---

Unlike rockchip android, we don't limit you how to get mac address. You can write/read it in any places. 

Usually, it could be stored in reserved1.  
http://opensource.rock-chips.com/wiki_Partitions

Bellow is how to burn mac address to reserved1 and read it.

First, enter maskrom and issue fllowing command to burn mac adress.

    printf '\xde\xad\xbe\xef\xef\xee' > mac.bin
    build/flash_tool.sh -p reserved1 -c rk3288 -i mac.bin

Then, add ethadder-set fucntion to u-boot board file, it will read mac address from reserved1.

    #include <mmc.h>

    int rk_board_late_init(void)
    {
      u8 ethaddr[1024];

      // 1 for sd-card
      struct mmc *mmc = find_mmc_device(0);
      struct blk_desc *desc = mmc_get_blk_desc(mmc);
      unsigned long blk_start = 8064;
      unsigned long blk_cnt = 1;

      int d = blk_dread(desc, blk_start, blk_cnt, ethaddr);

      eth_setenv_enetaddr("ethaddr", ethaddr);

      return 0;
    }
  
