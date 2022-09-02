---
toc: true
layout: post
comments: true
description: "Automation of reporting Part 2"
categories: [data analysis, mysql, gitlab, telegram, bot]
title: "Automation of reporting Part 2"
---
I continue my publications on analytics:

* [A Data Analysis in business - BI system and visualisation](https://alex.gladkikh.org/data%20analysis/mysql/superset/bi/2022/08/03/the-first-dashboard.html)
* [Analyses of Product Metrics](https://alex.gladkikh.org/data%20analysis/mysql/superset/bi/2022/08/05/analyses-of-product-metrics.html)
* [А/B-tests - Part 1/3(AA-test)](https://alex.gladkikh.org/data%20analysis/ab-test/aa-test/2022/08/09/aa-test-article.html)
* [А/B-tests - Part 2/3(AB-test)](https://alex.gladkikh.org/data%20analysis/ab-test/aa-test/2022/08/11/ab-test-article.html)
* [A/B-tests - Part 3/3 (relationship metrics)](https://alex.gladkikh.org/data%20analysis/ab-test/metrics/2022/08/15/ab-test-article-metrics.html)
* [Building an ETL-Pipeline (Airflow)](https://alex.gladkikh.org/data%20analysis/mysql/airflow/etl/2022/08/19/Building-ETL-Pipeline-Airflow.html)
* [Automation of reporting Part 1](https://alex.gladkikh.org/data%20analysis/mysql/gitlab/telegram/bot/2022/08/24/Automation-of-reporting.html)

Next, we needed to collect a single report on the operation of the entire application. The report should have included information on both the news feed and the message sending service. The report should arrive daily at 11:00 in the chat.

In the first step, we import all the necessary libraries and define the necessary variables. We also define the token of our bot, as we will need it for testing. Then we will hide it on GitLab, as we did in the last part:

~~~~python

import os
import telegram
import pandahouse as ph
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
import io
import pandas as pd

sns.set()

sns.set()
my_token = "**************************************"

chat_id = -1001592565485

connection = {
    'host': 'https://clickhouse.lab.karpov.courses',
    'password': '************',
    'user': '***********',
    'database': 'simulator_20220620'}
~~~~

Next, we form a text message that will be displayed in the report:

~~~~python

msg = """ 📃Application report for {date}📃
        Events: {events}
        🧑DAU: {users}({to_users_day_ago:+.2%} to day ago, {to_users_week_ago:+.2%} to week ago)
        🧑DAU by platform: 
            📱IOS: {users_ios}({to_users_ios_day_ago:+.2%} to day ago, {to_users_ios_week_ago:+.2%} to week ago)
            📞Android: {users_and}({to_users_and_day_ago:+.2%} to day ago, {to_users_and_week_ago:+.2%} to week ago)
        👫DAU by gender: 
            🙎Male: {users_male}({to_users_male_day_ago:+.2%} to day ago, {to_users_male_week_ago:+.2%} to week ago)
            👰‍Female: {users_female}({to_users_female_day_ago:+.2%} to day ago, {to_users_female_week_ago:+.2%} to week ago)
        🧑New users: {new_users}({to_new_users_day_ago:+.2%} to day ago, {to_new_users_week_ago:+.2%} to week ago)
        🎬Source:
            🍋ads: {new_users_ads}({to_new_users_ads_day_ago:+.2%} to day ago, {to_new_users_ads_week_ago:+.2%} to week ago)
            🍓organic: {new_users_org}({to_new_users_org_day_ago:+.2%} to day ago, {to_new_users_org_week_ago:+.2%} to week ago)
        
        FEEDS:
        🧑DAU: {users_feed}({to_users_feed_day_ago:+.2%} to day ago, {to_users_feed_week_ago:+.2%} to week ago)
        👍Likes: {likes}({to_likes_day_ago:+.2%} to day ago, {to_likes_week_ago:+.2%} to week ago)
        👀Views: {views}({to_views_day_ago:+.2%} to day ago, {to_views_week_ago:+.2%} to week ago)
        🌈CTR: {ctr:+.2%}({to_ctr_day_ago:+.2%} to day ago, {to_ctr_week_ago:+.2%} to week ago)
        
        MESSENGER:
        🧑DAU: {users_msg}({to_users_msg_day_ago:+.2%} to day ago, {to_users_msg_week_ago:+.2%} to week ago)
        📨Messages: {msg} ({to_msg_day_ago:+.2%} to day ago, {to_msg_week_ago:+.2%} to week ago)
        🪪Messages per users: {mpu:.2}({to_mpu_day_ago:+.2%} to day ago, {to_mpu_week_ago:+.2%} to week ago)
        """
~~~~

Now we need to write queries in order to get data for all these metrics. The first request will be a request for the feed:

~~~~python
query_feed_q = """
                select 
                    toDate(time) as date
                    ,count(distinct user_id) as users_feed
                    ,countIf(action='like') as likes
                    ,countIf(action='view') as views
                    ,countIf(action='like') / countIf(action='view') as CTR
                    ,likes+views as events
                from simulator_20220620.feed_actions
                where toDate(time) between today() - 8 and today() - 1
                group by date
                order by date
                """

data_feed = pd.DataFrame(ph.read_clickhouse(query=query_feed_q, connection=connection))
~~~~

The next request will be a request for a messenger:

~~~~python
query_msg_q = """
                  select 
                      toDate(time) as date
                      ,count(distinct user_id) as users_msg
                      ,count(user_id) as msg
                      , msg / users_msg as mpu
                  from simulator_20220620.message_actions
                  where toDate(time) between today() - 8 and today() - 1
                  group by date
                  order by date
                  """
data_msg = pd.DataFrame(ph.read_clickhouse(query=query_msg_q, connection=connection))
~~~~

Now there is a big query in general on the platform, where data will be required from two tables:

~~~~python
query_dau_all_q = """select date, 
                                uniqExact(user_id) as users, 
                                uniqExactIf(user_id, os='iOS') as users_ios, 
                                uniqExactIf(user_id, os='Android') as users_and, 
                                uniqExactIf(user_id, gender=1) as users_male,
                                uniqExactIf(user_id, gender=0) as users_female
                        from
                            (select distinct toDate(time) as date, user_id, os, gender
                            from simulator_20220620.feed_actions
                            where toDate(time) between today() - 8 and today() - 1
                            union all
                            select distinct toDate(time) as date, user_id, os, gender
                            from simulator_20220620.message_actions
                            where toDate(time) between today() - 8 and today() - 1) as t
                        group by date
                        order by date"""

data_dau_all = pd.DataFrame(ph.read_clickhouse(query=query_dau_all_q, connection=connection))
~~~~

Now it remains to find the number of new users. In this request, you need to find the minimum date for each user and count it as the registration date:

~~~~python
data_new_users_q = """
select date, 
        uniqExact(user_id) as new_user, 
        uniqExactIf(user_id, source='ads') as new_user_ads,
        uniqExactIf(user_id, source='organic') as new_user_organic
from(
    select user_id,
            source, 
            min(dt_reg) as date 
    from
        (select user_id, 
                min(toDate(time)) as dt_reg, 
                source
        from simulator_20220620.feed_actions
        where toDate(time) between today() - 8 and today() - 1
        group by user_id, source
        union all
        select user_id,
                min(toDate(time)) as dt_reg, 
                source
        from simulator_20220620.message_actions
        where toDate(time) between today() - 8 and today() - 1
        group by user_id, source) as t1
    group by user_id, source) as t2
group by date
where date between today() - 8 and today() - 1
"""
data_new_users = pd.DataFrame(ph.read_clickhouse(query=data_new_users_q, connection=connection))
~~~~

Next, we set variables to work with the date and correct the date and data types for all our dataframes:

~~~~python
data_feed['date'] = pd.to_datetime(data_feed['date']).dt.date
data_msq['date'] = pd.to_datetime(data_msq['date']).dt.date
data_dau_all['date'] = pd.to_datetime(data_dau_all['date']).dt.date
data_new_users['date'] = pd.to_datetime(data_new_users['date']).dt.date

data_feed = data_feed.astype({'users_feed': int, 'likes': int, 'views': int, 'events': int})
data_msq = data_msq.astype({'users_msg': int, 'msg': int})
data_dau_all = data_dau_all.astype({'users': int, 'users_ios': int, 'users_and': int, 'users_male': int, 'users_female': int})
data_new_users = data_new_users.astype({'new_user': int, 'new_user_ads': int, 'new_user_organic': int})

today = pd.Timestamp('now') - pd.DateOffset(days=1)
day_ago = today - pd.DateOffset(days=1)
week_ago = today - pd.DateOffset(days=7)
~~~~

Now fill in the text template:

~~~~python
report = msg.format(date=today.date(),
                        events=data_msg[data_msg['date'] == today.date()]['msg'].iloc[0]
                        + data_feed[data_feed['date'] == today.date()]['events'].iloc[0],
                        users=data_dau_all[data_dau_all['date'] == today.date()]['users'].iloc[0],
                        to_users_day_ago = (data_dau_all[data_dau_all['date'] == today.date()]['users'].iloc[0]
                                           - data_dau_all[data_dau_all['date'] == day_ago.date()]['users'].iloc[0])
                                           /data_dau_all[data_dau_all['date'] == day_ago.date()]['users'].iloc[0],
                        to_users_week_ago = (data_dau_all[data_dau_all['date'] == today.date()]['users'].iloc[0]
                                           - data_dau_all[data_dau_all['date'] == week_ago.date()]['users'].iloc[0])
                                           /data_dau_all[data_dau_all['date'] == week_ago.date()]['users'].iloc[0],

                        users_ios=data_dau_all[data_dau_all['date'] == today.date()]['users_ios'].iloc[0],
                        to_users_ios_day_ago=(data_dau_all[data_dau_all['date'] == today.date()]['users_ios'].iloc[0]
                                          - data_dau_all[data_dau_all['date'] == day_ago.date()]['users_ios'].iloc[0])
                                         / data_dau_all[data_dau_all['date'] == day_ago.date()]['users_ios'].iloc[0],
                        to_users_ios_week_ago=(data_dau_all[data_dau_all['date'] == today.date()]['users_ios'].iloc[0]
                                           - data_dau_all[data_dau_all['date'] == week_ago.date()]['users_ios'].iloc[0])
                                          / data_dau_all[data_dau_all['date'] == week_ago.date()]['users_ios'].iloc[0],

                        users_and=data_dau_all[data_dau_all['date'] == today.date()]['users_and'].iloc[0],
                        to_users_and_day_ago=(data_dau_all[data_dau_all['date'] == today.date()]['users_and'].iloc[0]
                                   - data_dau_all[data_dau_all['date'] == day_ago.date()]['users_and'].iloc[0])
                                  / data_dau_all[data_dau_all['date'] == day_ago.date()]['users_and'].iloc[0],
                        to_users_and_week_ago=(data_dau_all[data_dau_all['date'] == today.date()]['users_and'].iloc[0]
                                               -
                                               data_dau_all[data_dau_all['date'] == week_ago.date()]['users_and'].iloc[
                                                   0])
                                              / data_dau_all[data_dau_all['date'] == week_ago.date()]['users_and'].iloc[
                                                  0],

                        users_male=data_dau_all[data_dau_all['date'] == today.date()]['users_male'].iloc[0],
                        to_users_male_day_ago=(data_dau_all[data_dau_all['date'] == today.date()]['users_male'].iloc[0]
                                   - data_dau_all[data_dau_all['date'] == day_ago.date()]['users_male'].iloc[0])
                                  / data_dau_all[data_dau_all['date'] == day_ago.date()]['users_male'].iloc[0],
                        to_users_male_week_ago=(data_dau_all[data_dau_all['date'] == today.date()]['users_male'].iloc[0]
                                               - data_dau_all[data_dau_all['date'] == week_ago.date()]['users_male'].iloc[0])
                                              / data_dau_all[data_dau_all['date'] == week_ago.date()]['users_male'].iloc[0],

                        users_female=data_dau_all[data_dau_all['date'] == today.date()]['users_female'].iloc[0],
                        to_users_female_day_ago=(data_dau_all[data_dau_all['date'] == today.date()]['users_female'].iloc[0]
                                   - data_dau_all[data_dau_all['date'] == day_ago.date()]['users_female'].iloc[0])
                                  / data_dau_all[data_dau_all['date'] == day_ago.date()]['users_female'].iloc[0],
                        to_users_female_week_ago=(data_dau_all[data_dau_all['date'] == today.date()]['users_female'].iloc[0]
                                               -
                                               data_dau_all[data_dau_all['date'] == week_ago.date()]['users_female'].iloc[
                                                   0])
                                              / data_dau_all[data_dau_all['date'] == week_ago.date()]['users_female'].iloc[
                                                  0],

                        new_users=data_new_users[data_new_users['date'] == today.date()]['new_users'].iloc[0],
                        to_new_users_day_ago=(data_new_users[data_new_users['date'] == today.date()]['new_users'].iloc[0]
                                   - data_new_users[data_new_users['date'] == day_ago.date()]['new_users'].iloc[0])
                                  / data_new_users[data_new_users['date'] == day_ago.date()]['new_users'].iloc[0],
                        to_new_users_week_ago=(data_new_users[data_new_users['date'] == today.date()]['new_users'].iloc[0]
                                               -
                                               data_new_users[data_new_users['date'] == week_ago.date()]['new_users'].iloc[
                                                   0])
                                              / data_new_users[data_new_users['date'] == week_ago.date()]['new_users'].iloc[
                                                  0],

                        new_users_ads=data_new_users[data_new_users['date'] == today.date()]['new_users_ads'].iloc[0],
                        to_new_users_ads_day_ago=(data_new_users[data_new_users['date'] == today.date()]['new_users_ads'].iloc[0]
                                    - data_new_users[data_new_users['date'] == day_ago.date()]['new_users_ads'].iloc[0])
                                   / data_new_users[data_new_users['date'] == day_ago.date()]['new_users_ads'].iloc[0],
                        to_new_users_ads_week_ago=(data_new_users[data_new_users['date'] == today.date()]['new_users_ads'].iloc[0]
                                                - data_new_users[data_new_users['date'] == week_ago.date()][
                                                    'new_users_ads'].iloc[0])
                                               /
                                               data_new_users[data_new_users['date'] == week_ago.date()]['new_users_ads'].iloc[
                                                   0],

                        new_users_org=data_new_users[data_new_users['date'] == today.date()]['new_users_organic'].iloc[
                                           0],
                        to_new_users_org_day_ago=(data_new_users[data_new_users['date'] == today.date()]['new_users_organic'].iloc[0]
                                      - data_new_users[data_new_users['date'] == day_ago.date()]['new_users_organic'].iloc[
                                          0])
                                     / data_new_users[data_new_users['date'] == day_ago.date()]['new_users_organic'].iloc[0],
                        to_new_users_org_week_ago=(data_new_users[data_new_users['date'] == today.date()][
                                                         'new_users_organic'].iloc[0]
                                                     - data_new_users[data_new_users['date'] == week_ago.date()][
                                                         'new_users_organic'].iloc[0])
                                                    /
                                                    data_new_users[data_new_users['date'] == week_ago.date()][
                                                        'new_users_organic'].iloc[
                                                        0],

                        users_feed=data_feed[data_feed['date'] == today.date()]['users_feed'].iloc[0],
                        to_users_feed_day_ago=(data_feed[data_feed['date'] == today.date()]['users_feed'].iloc[0]
                                          - data_feed[data_feed['date'] == day_ago.date()]['users_feed'].iloc[0])
                                         / data_feed[data_feed['date'] == day_ago.date()]['users_feed'].iloc[0],
                        to_users_feed_week_ago=(data_feed[data_feed['date'] == today.date()]['users_feed'].iloc[0]
                                           - data_feed[data_feed['date'] == week_ago.date()]['users_feed'].iloc[0])
                                          / data_feed[data_feed['date'] == week_ago.date()]['users_feed'].iloc[0],

                        likes=data_feed[data_feed['date'] == today.date()]['likes'].iloc[0],
                        to_likes_day_ago=(data_feed[data_feed['date'] == today.date()]['likes'].iloc[0]
                                          - data_feed[data_feed['date'] == day_ago.date()]['likes'].iloc[0])
                                         / data_feed[data_feed['date'] == day_ago.date()]['likes'].iloc[0],
                        to_likes_week_ago=(data_feed[data_feed['date'] == today.date()]['likes'].iloc[0]
                                           - data_feed[data_feed['date'] == week_ago.date()]['likes'].iloc[0])
                                          / data_feed[data_feed['date'] == week_ago.date()]['likes'].iloc[0],

                        views=data_feed[data_feed['date'] == today.date()]['views'].iloc[0],
                        to_views_day_ago=(data_feed[data_feed['date'] == today.date()]['views'].iloc[0]
                                          - data_feed[data_feed['date'] == day_ago.date()]['views'].iloc[0])
                                         / data_feed[data_feed['date'] == day_ago.date()]['views'].iloc[0],
                        to_views_week_ago=(data_feed[data_feed['date'] == today.date()]['views'].iloc[0]
                                           - data_feed[data_feed['date'] == week_ago.date()]['views'].iloc[0])
                                          / data_feed[data_feed['date'] == week_ago.date()]['views'].iloc[0],

                        ctr=data_feed[data_feed['date'] == today.date()]['CTR'].iloc[0],
                        to_ctr_day_ago=(data_feed[data_feed['date'] == today.date()]['CTR'].iloc[0]
                                        - data_feed[data_feed['date'] == day_ago.date()]['CTR'].iloc[0])
                                       / data_feed[data_feed['date'] == day_ago.date()]['CTR'].iloc[0],
                        to_ctr_week_ago=(data_feed[data_feed['date'] == today.date()]['CTR'].iloc[0]
                                         - data_feed[data_feed['date'] == week_ago.date()]['CTR'].iloc[0])
                                        / data_feed[data_feed['date'] == week_ago.date()]['CTR'].iloc[0],

                        users_msg=data_msg[data_msg['date'] == today.date()]['users_msg'].iloc[0],
                        to_users_msg_day_ago=(data_msg[data_msg['date'] == today.date()]['users_msg'].iloc[0]
                                               - data_msg[data_msg['date'] == day_ago.date()]['users_msg'].iloc[0])
                                              / data_msg[data_msg['date'] == day_ago.date()]['users_msg'].iloc[0],
                        to_users_msg_week_ago=(data_msg[data_msg['date'] == today.date()]['users_msg'].iloc[0]
                                                - data_msg[data_msg['date'] == week_ago.date()]['users_msg'].iloc[0])
                                               / data_msg[data_msg['date'] == week_ago.date()]['users_msg'].iloc[0],

                        msg=data_msg[data_msg['date'] == today.date()]['msg'].iloc[0],
                        to_msg_day_ago=(data_msg[data_msg['date'] == today.date()]['msg'].iloc[0]
                                          - data_msg[data_msg['date'] == day_ago.date()]['msg'].iloc[0])
                                         / data_msg[data_msg['date'] == day_ago.date()]['msg'].iloc[0],
                        to_msg_week_ago=(data_msg[data_msg['date'] == today.date()]['msg'].iloc[0]
                                           - data_msg[data_msg['date'] == week_ago.date()]['msg'].iloc[0])
                                          / data_msg[data_msg['date'] == week_ago.date()]['msg'].iloc[0],

                        mpu=data_msg[data_msg['date'] == today.date()]['mpu'].iloc[0],
                        to_mpu_day_ago=(data_msg[data_msg['date'] == today.date()]['mpu'].iloc[0]
                                          - data_msg[data_msg['date'] == day_ago.date()]['mpu'].iloc[0])
                                         / data_msg[data_msg['date'] == day_ago.date()]['mpu'].iloc[0],
                        to_mpu_week_ago=(data_msg[data_msg['date'] == today.date()]['mpu'].iloc[0]
                                           - data_msg[data_msg['date'] == week_ago.date()]['mpu'].iloc[0])
                                          / data_msg[data_msg['date'] == week_ago.date()]['mpu'].iloc[0]
                        )
~~~~

Now we can write a function for graphs:

~~~~python
def get_plot(data_feed, data_msg, data_dau_all, data_new_users):
    data = pd.merge(data_feed, data_msg, on='date')
    data = pd.merge(data, data_dau_all, on='date')
    data = pd.merge(data, data_new_users, on='date')

    data['events_app'] = data['events']+data['msg']

    plt_obj_all = []

    fig, axes = plt.subplots(3, figsize = (10, 14))
    fig.suptitle('Statistics for the entire application for 7 days')
    app_dict = {0:{'y': ['events_app'], 'title': 'Events'},
                1:{'y': ['users', 'users_ios', 'users_and'], 'title': 'DAU'},
                2:{'y': ['users', 'new_users_ads', 'new_users_organic'], 'title': 'New users'}
    }

    for i in range(3):
        for y in app_dict[i]['y']:
            sns.lineplot(ax=axes[i], data=data, x='date', y=y)
        axes[i].set_title(app_dict[(i)]['title'])
        axes[i].set_xlabel(None)
        axes[i].set_ylabel(None)
        axes[i].legend(app_dict[i]['y'])
        for ind, label in enumerate(axes[i].get_xticklabels()):
            if ind % 3 == 0:
                label.set_visible(True)
            else:
                label.set_visible(False)

    plt_obj = io.BytesIO()
    plt.savefig(plt_obj)
    plt_obj.name = 'app_stat.png'
    plt_obj.seek(0)
    plt.close()
    plt_obj_all.append(plt_obj)
    #feed
    fig, axes = plt.subplots(2, 2, figsize=(15, 14))
    fig.suptitle('Statistics on the application feed for 7 days')
    plot_dict = {(0, 0): {'y': 'users_feed', 'title': 'Unique users'},
                 (0, 1): {'y': 'likes', 'title': 'Likes'},
                 (1, 0): {'y': 'views', 'title': 'Views'},
                 (1, 1): {'y': 'CTR', 'title': 'CTR'}}

    for i in range(2):
        for j in range(2):
            sns.lineplot(ax=axes[i, j], data=data, x='date', y=plot_dict[(i, j)]['y'])
            axes[i, j].set_title(plot_dict[(i, j)]['title'])
            axes[i, j].set_xlabel(None)
            axes[i, j].set_ylabel(None)
            for ind, label in enumerate(axes[i, j].get_xticklabels()):
                if ind % 3 == 0:
                    label.set_visible(True)
                else:
                    label.set_visible(False)

    plt_obj = io.BytesIO()
    plt.savefig(plt_obj)
    plt_obj.name = 'feed_stat.png'
    plt_obj.seek(0)
    plt.close()
    plt_obj_all.append(plt_obj)
    #messenger
    fig, axes = plt.subplots(3, figsize=(10, 14))
    fig.suptitle('Messenger app statistics for 7 days')
    msg_dict = {0: {'y': 'users_msg', 'title': 'DAU'},
                1: {'y': 'msg', 'title': 'Messages'},
                2: {'y': 'mpu', 'title': 'Messages per user'}
                }

    for i in range(3):
        sns.lineplot(ax=axes[i], data=data, x='date', y=msg_dict[i]['y'])
        axes[i].set_title(msg_dict[(i)]['title'])
        axes[i].set_xlabel(None)
        axes[i].set_ylabel(None)
        for ind, label in enumerate(axes[i].get_xticklabels()):
            if ind % 3 == 0:
                label.set_visible(True)
            else:
                label.set_visible(False)

    plt_obj = io.BytesIO()
    plt.savefig(plt_obj)
    plt_obj.name = 'msg_stat.png'
    plt_obj.seek(0)
    plt.close()
    plt_obj_all.append(plt_obj)
    return plt_obj_all
~~~~

After I wrote the function for the graph, I added the following lines to the end of the send_telegram_report function and called the function at the very end:

~~~~python
plt_obj = get_plot(data_feed, data_msg, data_dau_all, data_new_users)
bot.sendMessage(chat_id=chat_id, text=report)
for pl in plt_obj:
    bot.sendPhoto(chat_id=chat_id, photo=pl)
~~~~


~~~~python
send_telegram_report(chat_id)
~~~~

Upload the resulting file to any system for automation. As in the last article, I will use GitLab CI/CD. I already have a task for automation there. In this connection, I will create a second task and assign variables to both tasks so that it is possible to automate them in the yml file. Variables should be assigned for the old and new tasks.

![image](https://raw.githubusercontent.com/Zmey56/blog/master/_posts/images/Automation_reporting_2/tel_1.png)

![image](https://raw.githubusercontent.com/Zmey56/blog/master/_posts/images/Automation_reporting_2/tel_2.png)

After that, I make changes to the yml file and that's it. Reports will be sent to our group daily at 11.00 and 11.05:

~~~~
image: cr.yandex/crp742p3qacifd2hcon2/practice-da:latest

stages:
  - init
  - run


feed_report_job:
    stage: run
    script:
        - python main.py
    only:
      refs:
        - schedules
      variables:
        - $SCHEDULE_TYPE == "build_feed_report"

app_report_job:
    stage: run
    script:
        - python app_report.py
    only:
      refs:
        - schedules
      variables:
        - $SCHEDULE_TYPE == "build_app_report"
~~~~

The reports in my group as a result of the operation of this algorithm look like this:

![image](https://raw.githubusercontent.com/Zmey56/blog/master/_posts/images/Automation_reporting_2/tel_3.png)

![image](https://raw.githubusercontent.com/Zmey56/blog/master/_posts/images/Automation_reporting_2/tel_4.png)

![image](https://raw.githubusercontent.com/Zmey56/blog/master/_posts/images/Automation_reporting_2/tel_5.png)

![image](https://raw.githubusercontent.com/Zmey56/blog/master/_posts/images/Automation_reporting_2/tel_6.png)


As a result of the work done, we managed to create an automated reporting system that provides daily information to the corporate krupp in the telegram channel. In the following final articles, I will review the work in alerts.

Full code:
~~~~python

import os
import telegram
import pandahouse as ph
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
import io
import pandas as pd

sns.set()
bot = telegram.Bot(token=os.environ.get("REPORT_BOT_TOKEN"))

chat_id = -1001592565485

connection = {
    'host': 'https://clickhouse.lab.karpov.courses',
    'password': 'dpo_python_2020',
    'user': 'student',
    'database': 'simulator_20220620'}

def get_plot(data_feed, data_msg, data_dau_all, data_new_users):
    data = pd.merge(data_feed, data_msg, on='date')
    data = pd.merge(data, data_dau_all, on='date')
    data = pd.merge(data, data_new_users, on='date')

    data['events_app'] = data['events']+data['msg']

    plt_obj_all = []

    fig, axes = plt.subplots(3, figsize = (10, 14))
    fig.suptitle('Statistics for the entire application for 7 days')
    app_dict = {0:{'y': ['events_app'], 'title': 'Events'},
                1:{'y': ['users', 'users_ios', 'users_and'], 'title': 'DAU'},
                2:{'y': ['users', 'new_users_ads', 'new_users_organic'], 'title': 'New users'}
    }

    for i in range(3):
        for y in app_dict[i]['y']:
            sns.lineplot(ax=axes[i], data=data, x='date', y=y)
        axes[i].set_title(app_dict[(i)]['title'])
        axes[i].set_xlabel(None)
        axes[i].set_ylabel(None)
        axes[i].legend(app_dict[i]['y'])
        for ind, label in enumerate(axes[i].get_xticklabels()):
            if ind % 3 == 0:
                label.set_visible(True)
            else:
                label.set_visible(False)

    plt_obj = io.BytesIO()
    plt.savefig(plt_obj)
    plt_obj.name = 'app_stat.png'
    plt_obj.seek(0)
    plt.close()
    plt_obj_all.append(plt_obj)
    #feed
    fig, axes = plt.subplots(2, 2, figsize=(15, 14))
    fig.suptitle('Statistics on the application feed for 7 days')
    plot_dict = {(0, 0): {'y': 'users_feed', 'title': 'Unique users'},
                 (0, 1): {'y': 'likes', 'title': 'Likes'},
                 (1, 0): {'y': 'views', 'title': 'Views'},
                 (1, 1): {'y': 'CTR', 'title': 'CTR'}}

    for i in range(2):
        for j in range(2):
            sns.lineplot(ax=axes[i, j], data=data, x='date', y=plot_dict[(i, j)]['y'])
            axes[i, j].set_title(plot_dict[(i, j)]['title'])
            axes[i, j].set_xlabel(None)
            axes[i, j].set_ylabel(None)
            for ind, label in enumerate(axes[i, j].get_xticklabels()):
                if ind % 3 == 0:
                    label.set_visible(True)
                else:
                    label.set_visible(False)

    plt_obj = io.BytesIO()
    plt.savefig(plt_obj)
    plt_obj.name = 'feed_stat.png'
    plt_obj.seek(0)
    plt.close()
    plt_obj_all.append(plt_obj)
    #messenger
    fig, axes = plt.subplots(3, figsize=(10, 14))
    fig.suptitle('Messenger app statistics for 7 days')
    msg_dict = {0: {'y': 'users_msg', 'title': 'DAU'},
                1: {'y': 'msg', 'title': 'Messages'},
                2: {'y': 'mpu', 'title': 'Messages per user'}
                }

    for i in range(3):
        sns.lineplot(ax=axes[i], data=data, x='date', y=msg_dict[i]['y'])
        axes[i].set_title(msg_dict[(i)]['title'])
        axes[i].set_xlabel(None)
        axes[i].set_ylabel(None)
        for ind, label in enumerate(axes[i].get_xticklabels()):
            if ind % 3 == 0:
                label.set_visible(True)
            else:
                label.set_visible(False)

    plt_obj = io.BytesIO()
    plt.savefig(plt_obj)
    plt_obj.name = 'msg_stat.png'
    plt_obj.seek(0)
    plt.close()
    plt_obj_all.append(plt_obj)
    return plt_obj_all

def send_telegram_report(chat):
    chat = chat or chat_id
    msg = """ 📃Application report for {date}📃
        Events: {events}
        🧑DAU: {users}({to_users_day_ago:+.2%} to day ago, {to_users_week_ago:+.2%} to week ago)
        🧑DAU by platform: 
            📱IOS: {users_ios}({to_users_ios_day_ago:+.2%} to day ago, {to_users_ios_week_ago:+.2%} to week ago)
            📞Android: {users_and}({to_users_and_day_ago:+.2%} to day ago, {to_users_and_week_ago:+.2%} to week ago)
        👫DAU by gender: 
            🙎Male: {users_male}({to_users_male_day_ago:+.2%} to day ago, {to_users_male_week_ago:+.2%} to week ago)
            👰‍Female: {users_female}({to_users_female_day_ago:+.2%} to day ago, {to_users_female_week_ago:+.2%} to week ago)
        🧑New users: {new_users}({to_new_users_day_ago:+.2%} to day ago, {to_new_users_week_ago:+.2%} to week ago)
        🎬Source:
            🍋ads: {new_users_ads}({to_new_users_ads_day_ago:+.2%} to day ago, {to_new_users_ads_week_ago:+.2%} to week ago)
            🍓organic: {new_users_org}({to_new_users_org_day_ago:+.2%} to day ago, {to_new_users_org_week_ago:+.2%} to week ago)
        
        FEEDS:
        🧑DAU: {users_feed}({to_users_feed_day_ago:+.2%} to day ago, {to_users_feed_week_ago:+.2%} to week ago)
        👍Likes: {likes}({to_likes_day_ago:+.2%} to day ago, {to_likes_week_ago:+.2%} to week ago)
        👀Views: {views}({to_views_day_ago:+.2%} to day ago, {to_views_week_ago:+.2%} to week ago)
        🌈CTR: {ctr:+.2%}({to_ctr_day_ago:+.2%} to day ago, {to_ctr_week_ago:+.2%} to week ago)
        
        MESSENGER:
        🧑DAU: {users_msg}({to_users_msg_day_ago:+.2%} to day ago, {to_users_msg_week_ago:+.2%} to week ago)
        📨Messages: {msg} ({to_msg_day_ago:+.2%} to day ago, {to_msg_week_ago:+.2%} to week ago)
        🪪Messages per users: {mpu:.2}({to_mpu_day_ago:+.2%} to day ago, {to_mpu_week_ago:+.2%} to week ago)
        """

    query_feed_q = """
                select 
                    toDate(time) as date
                    ,count(distinct user_id) as users_feed
                    ,countIf(action='like') as likes
                    ,countIf(action='view') as views
                    ,countIf(action='like') / countIf(action='view') as CTR
                    ,likes+views as events
                from simulator_20220620.feed_actions
                where toDate(time) between today() - 8 and today() - 1
                group by date
                order by date
                """

    data_feed = pd.DataFrame(ph.read_clickhouse(query=query_feed_q, connection=connection))

    query_msg_q = """
                  select 
                      toDate(time) as date
                      ,count(distinct user_id) as users_msg
                      ,count(user_id) as msg
                      , msg / users_msg as mpu
                  from simulator_20220620.message_actions
                  where toDate(time) between today() - 8 and today() - 1
                  group by date
                  order by date
                  """
    data_msg = pd.DataFrame(ph.read_clickhouse(query=query_msg_q, connection=connection))

    query_dau_all_q = """select date, 
                                uniqExact(user_id) as users, 
                                uniqExactIf(user_id, os='iOS') as users_ios, 
                                uniqExactIf(user_id, os='Android') as users_and, 
                                uniqExactIf(user_id, gender=1) as users_male,
                                uniqExactIf(user_id, gender=0) as users_female
                        from
                            (select distinct toDate(time) as date, user_id, os, gender
                            from simulator_20220620.feed_actions
                            where toDate(time) between today() - 8 and today() - 1
                            union all
                            select distinct toDate(time) as date, user_id, os, gender
                            from simulator_20220620.message_actions
                            where toDate(time) between today() - 8 and today() - 1) as t
                        group by date
                        order by date"""

    data_dau_all = pd.DataFrame(ph.read_clickhouse(query=query_dau_all_q, connection=connection))

    data_new_users_q = """
    select date, 
            uniqExact(user_id) as new_users, 
            uniqExactIf(user_id, source='ads') as new_users_ads,
            uniqExactIf(user_id, source='organic') as new_users_organic
    from(
        select user_id,
                source, 
                min(dt_reg) as date 
        from
            (select user_id, 
                    min(toDate(time)) as dt_reg, 
                    source
            from simulator_20220620.feed_actions
            where toDate(time) between today() - 8 and today() - 1
            group by user_id, source
            union all
            select user_id,
                    min(toDate(time)) as dt_reg, 
                    source
            from simulator_20220620.message_actions
            where toDate(time) between today() - 8 and today() - 1
            group by user_id, source) as t1
        group by user_id, source) as t2
    where date between today() - 8 and today() - 1
    group by date
    """
    data_new_users = pd.DataFrame(ph.read_clickhouse(query=data_new_users_q, connection=connection))

    today = pd.Timestamp('now') - pd.DateOffset(days=1)
    day_ago = today - pd.DateOffset(days=1)
    week_ago = today - pd.DateOffset(days=7)

    data_feed['date'] = pd.to_datetime(data_feed['date']).dt.date
    data_msg['date'] = pd.to_datetime(data_msg['date']).dt.date
    data_dau_all['date'] = pd.to_datetime(data_dau_all['date']).dt.date
    data_new_users['date'] = pd.to_datetime(data_new_users['date']).dt.date

    data_feed = data_feed.astype({'users_feed': int, 'likes': int, 'views': int, 'events': int})
    data_msg = data_msg.astype({'users_msg': int, 'msg': int})
    data_dau_all = data_dau_all.astype({'users': int, 'users_ios': int, 'users_and': int, 'users_male': int, 'users_female': int})
    data_new_users = data_new_users.astype({'new_users': int, 'new_users_ads': int, 'new_users_organic': int})

    report = msg.format(date=today.date(),
                        events=data_msg[data_msg['date'] == today.date()]['msg'].iloc[0]
                        + data_feed[data_feed['date'] == today.date()]['events'].iloc[0],
                        users=data_dau_all[data_dau_all['date'] == today.date()]['users'].iloc[0],
                        to_users_day_ago = (data_dau_all[data_dau_all['date'] == today.date()]['users'].iloc[0]
                                           - data_dau_all[data_dau_all['date'] == day_ago.date()]['users'].iloc[0])
                                           /data_dau_all[data_dau_all['date'] == day_ago.date()]['users'].iloc[0],
                        to_users_week_ago = (data_dau_all[data_dau_all['date'] == today.date()]['users'].iloc[0]
                                           - data_dau_all[data_dau_all['date'] == week_ago.date()]['users'].iloc[0])
                                           /data_dau_all[data_dau_all['date'] == week_ago.date()]['users'].iloc[0],

                        users_ios=data_dau_all[data_dau_all['date'] == today.date()]['users_ios'].iloc[0],
                        to_users_ios_day_ago=(data_dau_all[data_dau_all['date'] == today.date()]['users_ios'].iloc[0]
                                          - data_dau_all[data_dau_all['date'] == day_ago.date()]['users_ios'].iloc[0])
                                         / data_dau_all[data_dau_all['date'] == day_ago.date()]['users_ios'].iloc[0],
                        to_users_ios_week_ago=(data_dau_all[data_dau_all['date'] == today.date()]['users_ios'].iloc[0]
                                           - data_dau_all[data_dau_all['date'] == week_ago.date()]['users_ios'].iloc[0])
                                          / data_dau_all[data_dau_all['date'] == week_ago.date()]['users_ios'].iloc[0],

                        users_and=data_dau_all[data_dau_all['date'] == today.date()]['users_and'].iloc[0],
                        to_users_and_day_ago=(data_dau_all[data_dau_all['date'] == today.date()]['users_and'].iloc[0]
                                   - data_dau_all[data_dau_all['date'] == day_ago.date()]['users_and'].iloc[0])
                                  / data_dau_all[data_dau_all['date'] == day_ago.date()]['users_and'].iloc[0],
                        to_users_and_week_ago=(data_dau_all[data_dau_all['date'] == today.date()]['users_and'].iloc[0]
                                               -
                                               data_dau_all[data_dau_all['date'] == week_ago.date()]['users_and'].iloc[
                                                   0])
                                              / data_dau_all[data_dau_all['date'] == week_ago.date()]['users_and'].iloc[
                                                  0],

                        users_male=data_dau_all[data_dau_all['date'] == today.date()]['users_male'].iloc[0],
                        to_users_male_day_ago=(data_dau_all[data_dau_all['date'] == today.date()]['users_male'].iloc[0]
                                   - data_dau_all[data_dau_all['date'] == day_ago.date()]['users_male'].iloc[0])
                                  / data_dau_all[data_dau_all['date'] == day_ago.date()]['users_male'].iloc[0],
                        to_users_male_week_ago=(data_dau_all[data_dau_all['date'] == today.date()]['users_male'].iloc[0]
                                               - data_dau_all[data_dau_all['date'] == week_ago.date()]['users_male'].iloc[0])
                                              / data_dau_all[data_dau_all['date'] == week_ago.date()]['users_male'].iloc[0],

                        users_female=data_dau_all[data_dau_all['date'] == today.date()]['users_female'].iloc[0],
                        to_users_female_day_ago=(data_dau_all[data_dau_all['date'] == today.date()]['users_female'].iloc[0]
                                   - data_dau_all[data_dau_all['date'] == day_ago.date()]['users_female'].iloc[0])
                                  / data_dau_all[data_dau_all['date'] == day_ago.date()]['users_female'].iloc[0],
                        to_users_female_week_ago=(data_dau_all[data_dau_all['date'] == today.date()]['users_female'].iloc[0]
                                               -
                                               data_dau_all[data_dau_all['date'] == week_ago.date()]['users_female'].iloc[
                                                   0])
                                              / data_dau_all[data_dau_all['date'] == week_ago.date()]['users_female'].iloc[
                                                  0],

                        new_users=data_new_users[data_new_users['date'] == today.date()]['new_users'].iloc[0],
                        to_new_users_day_ago=(data_new_users[data_new_users['date'] == today.date()]['new_users'].iloc[0]
                                   - data_new_users[data_new_users['date'] == day_ago.date()]['new_users'].iloc[0])
                                  / data_new_users[data_new_users['date'] == day_ago.date()]['new_users'].iloc[0],
                        to_new_users_week_ago=(data_new_users[data_new_users['date'] == today.date()]['new_users'].iloc[0]
                                               -
                                               data_new_users[data_new_users['date'] == week_ago.date()]['new_users'].iloc[
                                                   0])
                                              / data_new_users[data_new_users['date'] == week_ago.date()]['new_users'].iloc[
                                                  0],

                        new_users_ads=data_new_users[data_new_users['date'] == today.date()]['new_users_ads'].iloc[0],
                        to_new_users_ads_day_ago=(data_new_users[data_new_users['date'] == today.date()]['new_users_ads'].iloc[0]
                                    - data_new_users[data_new_users['date'] == day_ago.date()]['new_users_ads'].iloc[0])
                                   / data_new_users[data_new_users['date'] == day_ago.date()]['new_users_ads'].iloc[0],
                        to_new_users_ads_week_ago=(data_new_users[data_new_users['date'] == today.date()]['new_users_ads'].iloc[0]
                                                - data_new_users[data_new_users['date'] == week_ago.date()][
                                                    'new_users_ads'].iloc[0])
                                               /
                                               data_new_users[data_new_users['date'] == week_ago.date()]['new_users_ads'].iloc[
                                                   0],

                        new_users_org=data_new_users[data_new_users['date'] == today.date()]['new_users_organic'].iloc[
                                           0],
                        to_new_users_org_day_ago=(data_new_users[data_new_users['date'] == today.date()]['new_users_organic'].iloc[0]
                                      - data_new_users[data_new_users['date'] == day_ago.date()]['new_users_organic'].iloc[
                                          0])
                                     / data_new_users[data_new_users['date'] == day_ago.date()]['new_users_organic'].iloc[0],
                        to_new_users_org_week_ago=(data_new_users[data_new_users['date'] == today.date()][
                                                         'new_users_organic'].iloc[0]
                                                     - data_new_users[data_new_users['date'] == week_ago.date()][
                                                         'new_users_organic'].iloc[0])
                                                    /
                                                    data_new_users[data_new_users['date'] == week_ago.date()][
                                                        'new_users_organic'].iloc[
                                                        0],

                        users_feed=data_feed[data_feed['date'] == today.date()]['users_feed'].iloc[0],
                        to_users_feed_day_ago=(data_feed[data_feed['date'] == today.date()]['users_feed'].iloc[0]
                                          - data_feed[data_feed['date'] == day_ago.date()]['users_feed'].iloc[0])
                                         / data_feed[data_feed['date'] == day_ago.date()]['users_feed'].iloc[0],
                        to_users_feed_week_ago=(data_feed[data_feed['date'] == today.date()]['users_feed'].iloc[0]
                                           - data_feed[data_feed['date'] == week_ago.date()]['users_feed'].iloc[0])
                                          / data_feed[data_feed['date'] == week_ago.date()]['users_feed'].iloc[0],

                        likes=data_feed[data_feed['date'] == today.date()]['likes'].iloc[0],
                        to_likes_day_ago=(data_feed[data_feed['date'] == today.date()]['likes'].iloc[0]
                                          - data_feed[data_feed['date'] == day_ago.date()]['likes'].iloc[0])
                                         / data_feed[data_feed['date'] == day_ago.date()]['likes'].iloc[0],
                        to_likes_week_ago=(data_feed[data_feed['date'] == today.date()]['likes'].iloc[0]
                                           - data_feed[data_feed['date'] == week_ago.date()]['likes'].iloc[0])
                                          / data_feed[data_feed['date'] == week_ago.date()]['likes'].iloc[0],

                        views=data_feed[data_feed['date'] == today.date()]['views'].iloc[0],
                        to_views_day_ago=(data_feed[data_feed['date'] == today.date()]['views'].iloc[0]
                                          - data_feed[data_feed['date'] == day_ago.date()]['views'].iloc[0])
                                         / data_feed[data_feed['date'] == day_ago.date()]['views'].iloc[0],
                        to_views_week_ago=(data_feed[data_feed['date'] == today.date()]['views'].iloc[0]
                                           - data_feed[data_feed['date'] == week_ago.date()]['views'].iloc[0])
                                          / data_feed[data_feed['date'] == week_ago.date()]['views'].iloc[0],

                        ctr=data_feed[data_feed['date'] == today.date()]['CTR'].iloc[0],
                        to_ctr_day_ago=(data_feed[data_feed['date'] == today.date()]['CTR'].iloc[0]
                                        - data_feed[data_feed['date'] == day_ago.date()]['CTR'].iloc[0])
                                       / data_feed[data_feed['date'] == day_ago.date()]['CTR'].iloc[0],
                        to_ctr_week_ago=(data_feed[data_feed['date'] == today.date()]['CTR'].iloc[0]
                                         - data_feed[data_feed['date'] == week_ago.date()]['CTR'].iloc[0])
                                        / data_feed[data_feed['date'] == week_ago.date()]['CTR'].iloc[0],

                        users_msg=data_msg[data_msg['date'] == today.date()]['users_msg'].iloc[0],
                        to_users_msg_day_ago=(data_msg[data_msg['date'] == today.date()]['users_msg'].iloc[0]
                                               - data_msg[data_msg['date'] == day_ago.date()]['users_msg'].iloc[0])
                                              / data_msg[data_msg['date'] == day_ago.date()]['users_msg'].iloc[0],
                        to_users_msg_week_ago=(data_msg[data_msg['date'] == today.date()]['users_msg'].iloc[0]
                                                - data_msg[data_msg['date'] == week_ago.date()]['users_msg'].iloc[0])
                                               / data_msg[data_msg['date'] == week_ago.date()]['users_msg'].iloc[0],

                        msg=data_msg[data_msg['date'] == today.date()]['msg'].iloc[0],
                        to_msg_day_ago=(data_msg[data_msg['date'] == today.date()]['msg'].iloc[0]
                                          - data_msg[data_msg['date'] == day_ago.date()]['msg'].iloc[0])
                                         / data_msg[data_msg['date'] == day_ago.date()]['msg'].iloc[0],
                        to_msg_week_ago=(data_msg[data_msg['date'] == today.date()]['msg'].iloc[0]
                                           - data_msg[data_msg['date'] == week_ago.date()]['msg'].iloc[0])
                                          / data_msg[data_msg['date'] == week_ago.date()]['msg'].iloc[0],

                        mpu=data_msg[data_msg['date'] == today.date()]['mpu'].iloc[0],
                        to_mpu_day_ago=(data_msg[data_msg['date'] == today.date()]['mpu'].iloc[0]
                                          - data_msg[data_msg['date'] == day_ago.date()]['mpu'].iloc[0])
                                         / data_msg[data_msg['date'] == day_ago.date()]['mpu'].iloc[0],
                        to_mpu_week_ago=(data_msg[data_msg['date'] == today.date()]['mpu'].iloc[0]
                                           - data_msg[data_msg['date'] == week_ago.date()]['mpu'].iloc[0])
                                          / data_msg[data_msg['date'] == week_ago.date()]['mpu'].iloc[0]
                        )
    plt_obj = get_plot(data_feed, data_msg, data_dau_all, data_new_users)
    bot.sendMessage(chat_id=chat_id, text=report)
    for pl in plt_obj:
        bot.sendPhoto(chat_id=chat_id, photo=pl)


send_telegram_report(chat_id)

~~~~
