---
layout: post
title: Rails with sidekiq-schduler 實作 cron jobs 設定（施工中...）
date: 2022-02-24 18:11 +0800
tags: update rails
---

### Implementations

- sidekiq-cron → sidekiq-schedule
    - [How to create scheduled jobs using sidekiq-cron](https://medium.com/geekculture/how-to-create-scheduled-jobs-in-rails-using-sidekiq-cron-dc5dee27eae5)
    - https://github.com/moove-it/sidekiq-scheduler
- ActiveJobs

### Config

- 使用 Overmind 一次啟動 Web/Sidekiq/Webpacker（Overmind 可以用 byebug）
- 開啟所有 server: `OVERMIND_PORT=3000 overmind start -f Procfile.dev`
- `Redis::CannotConnectError: Error connecting to Redis on 127.0.0.1:6379 (Errno::ECONNREFUSED)` → 開 redis server  `redis-server --daemonize yes`

### Problems

- 每個門診的報導截止時間是動態的
    - ~~新增 hospital~~ 或修改門診時段時，檢查有要執行檢查的時間列表(`Sidekiq.get_schedule`) ，沒有在列表裡的話就新增 cron job → 應該是只針對一家醫院
- ActiveRecord/ActiveModel**::Dirty**
    - ~~到底為啥改不動😀🔪~~
    - ****`saved_change_to_attribute(attr_name)`****
    - `name_changed?`
- Difference between service and job
    - service: 簡化程式、重複使用
    - job: 執行任務用
- Job 內容要寫啥
    - 找到報到時間截止為現在的所有 `hospital`
        - 存門診時間的欄位是 `[jsonb](https://nandovieira.com/using-postgresql-and-jsonb-with-ruby-on-rails)`，query 的方式不同

            ```ruby
            Hospital.where('clinic_preference @> ?', {close_booking_on_for_locale_first: -30}.to_json)
            ```

        - SQL

            ```sql
            SELECT * FROM Hospitals WHERE (SELECT (Hospitals.clinic_preferences->>'morning_starts_at')::int+(Hospitals.clinic_preferences->>'close_checkin_for_first')::int*60 FROM Hospitals) = 39600;
            ```

        - ORM

            ```ruby
            hospitals = Hospital.where("(Hospitals.clinic_preferences->>'morning_starts_at')::int
                                       +(Hospitals.clinic_preferences->>'close_checkin_for_first')::int*60 = ?", now)
            ```

    - 報到截止時間根據門診時間有早、午、晚的時段差異，還要處理初複診的身份差異，**而且刪除的必須要是當下門診的掛號們，不能刪到其他正常未報到掛號**

        從 hospital 下去找 booking（限制時段與日期）

        1. 找到報到截止時間為現在的 Hospitals（如果找不到結果就跳到下一層回圈）
        2. 對找到的 hospitals 尋找當日掛號，條件：狀態 `booked`、日期 `Date.current`、時段 `period`（從第一層 query 來的）
        3. 將找到的這些 bookings 狀態切換為 `cancelled`

---

0209: ？？？等等我好像寫好了？？明天想一下要怎麼測試&加上從 preference controller 判別

---

### Sidekiq-scheduler

- Sidekiq webui: `https://localhost:3000/sidekiq`
    - 帳號密碼：admin/admin
- ~~新增 initializer: `config/initializers/sidekiq_scheduler.rb` [[Ref]](https://github.com/moove-it/sidekiq-scheduler#manage-tasks-from-unicornrails-server)~~  → 這是沒有用 sidekiq 的情況下要用 sidekiq-scheduler 的寫法
- sidekiq 內容跟 rails job 還沒有接起來（sidekiq log 上有 cron job 的紀錄）
    - 要在 `application.rb`  裡設定：`config.active_job.queue_adapter = :sidekiq`

設定成功囉！

---

### CancelUnvisitedJob + 測試

- Rspec `around` block: 可以將要執行測試的環境獨立出來，例如 `travel_to`
- 不知道為啥測試一直寫不過，在網頁上跑明明就是正確的 → 在改動完資料庫內容之後要執行 `reload`（他不會自動更新）
