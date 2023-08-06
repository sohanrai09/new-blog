---
title: "Network TUI"
date: 2023-08-03T18:28:20+05:30
categories:
- textual
- python
#- subcategory
tags:
- python
- textual
# - tag2
keywords:
- tech
thumbnailImagePosition: left
thumbnailImage: "https://images.pexels.com/photos/207580/pexels-photo-207580.jpeg"
coverImage: "https://images.pexels.com/photos/207580/pexels-photo-207580.jpeg"
---


At the end of my previous [blog post](https://sohanrai09.github.io/new-blog/2023/04/network-dashboard/) I mentioned about wanting to work on [Textual](https://textual.textualize.io/), and I'm excited to share here my first project using Textual. I feel like for us network engineers, working on a CLI makes our work more enjoyable so naturally, building a TUI(Terminal User Interface) to do some of the automation tasks was a no-brainer to me. I have called it `Net-TUI`, a TUI application to do few network automation tasks.

### Net-TUI

As mentioned earlier, I'm using `Textual` to build the TUI App and to help me with networking tasks, I'm using [Nornir](https://nornir.readthedocs.io/en/latest/). `Net-TUI` is broadly split into three sections. 1.Dashboard 2.Checker and 3.Generator. I will be going through each of these in the following sections.

### Dashboard

To anyone following my blog, you would know that I went over building a Networking Dashboard in the previous [blog post](https://sohanrai09.github.io/new-blog/2023/04/network-dashboard/). I'm using it as a reference here with some enhancements and user interactions to make it more 'App like'. Below is the snippet of how it looks like when the App is launched. As you can see, user has an option to provide the `device name` of which they want to build the Dashboard off. `device name` input comes with a nice little `dropdown` and `autocomplete` feature, which in this case is the list of available devices from the Nornir Inventory, which gets 'autocompleted' as we type along.

![tui_dash1](/images/tui_dash1.png)

Without going into more details of the data shown in Dashboard, as I have explained it my previous blog post, I would like to highlight few enhancements made to it. Unlike the previous version of Dashboard where the routing protocols were pre defined, in this version, there is an intelligence built to identify the protocols currently configured on the device and then build the Dashboard around that. As of now this is limited to some of the well-known protocols like BGP, ISIS, OSPF, MPLS, LDP but this can be easily extended to include anything else. Below snippet shows the end result of Dashboard.

![tui_dash2](/images/tui_dash2.png)


### Checker

This section has three sub-categories, each with an interesting function associated with them.

1. #### Card Look-up

   As the name suggests, this function can be used to look-up any particular card/FPC across the inventory of hosts as defined in Nornir. Input for card name again comes with a `dropdown` and `autocomplete` feature. Any matches to the card will be displayed along with the device name and FPC Slot information. If no matches are found, same will be displayed to the user. I found this feature to be quite handy as I have to deal with a huge inventory of devices at work and this gives a quick and easy way of knowing the cards deployed currently in the network. List of cards which is used to provide the `dropdown` options currently comes from a static file but a future enhancement could be to retrieve that information dynamically from Juniper/vendor websites.

   ![tui_card](/images/tui_card.png)

2. #### Config search

   This function can be used to quickly scan through the configuration of all the devices for any particular pattern match. Let's assume we want to check if a particular user is configured on all the devices or if we have RE(Routing Engine) Filter configured on all devices, this can help with these tasks and many more. Any matches found in the configuration will be displayed along with the device name.

   ![tui_config](/images/tui_config.png)

3. #### Fetching Command output

   This function helps in retrieving the output of any given CLI command from all the devices. Again, the available commands are displayed as `dropdown` with `autocomplete`feature. List of commands comes from a static file, so one can add any commands they would like to offer the user as an option to execute. Output of the commands will be displayed along with the device name.

   ![tui_cmd_out](/images/tui_cmd_out.png)


### Generator

My inspiration to this section comes from my days at Network Ops where every other day I used to work on some maintenance activities and with this comes the crucial part of pre and post network validation. Since each device we work on can be different, we could not always have the same checks for validation. Generator solves this problem by generating a list of commands as per the protocols configured on the device. User also has an option to select if they would want `terse` or `verbose` level of commands and the commands are generated accordingly, with `terse` being the default. Commands for each protocols are pre defined in a `YAML` file and one can make any changes to it as per their business needs. Once the commands are generated, user can either copy it to a clipboard using `Copy to clipboard` button at the `Footer` of the App or by simply using `CTRL+C`. A pop-up notification is generated to confirm that commands are copied to the clipboard. But better yet, if the user wishes to run the commands and fetch the output from the device, they can use `Fetch output` or `CTRL+F`. The App would then connect to the device and execute the commands and store the output in a .txt file, which will be displayed to the user at the end.

![tui_gen_out1](/images/tui_gen_out1.png)

![tui_gen_out2](/images/tui_gen_out2.png)


I honestly think that this blog post, with the screenshots, doesn't do justice to the TUI App. Please check out the screen recording [here](https://github.com/sohanrai09/net_tui/assets/89385413/38decade-628d-4414-91ac-6b748ee37be4)
to truly appreciate the beauty of Textual. This probably is the simplest version of TUIs out there but hopefully it goes to show what's possible with [Textual](https://textual.textualize.io/). Folks at Textual are adding new and interesting features regularly which gives us users more options to play with to customize our TUI Apps. This project has been the most fun filled projects I have worked on and hopefully I've been able to capture and present that here. Please feel free to reach out to me in case you need any further clarifications, code repository can be found [here](https://github.com/sohanrai09/net_tui). Checkout another great Network TUI built by [Danny Wade](https://twitter.com/devnetdan?s=20) called [net-textorial](https://github.com/dannywade/net-textorial)

Thank you!



