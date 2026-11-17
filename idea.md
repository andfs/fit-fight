# FitFight

FitFight is a fitness application designed to help users achieve their health and fitness goals through gamefication and social interaction. The app reads data from strava and various fitness trackers to calculate points based on the user's activities, weight loss and bioimpedance evolution. Users can compete with friends, join challenges, and track their progress over time.

## Features
- **Activity Tracking**: Sync with Strava and other fitness trackers to automatically log workouts and activities.
- **Point System**: Earn points for various activities, weight loss, and improvements in bioimpedance metrics. Points can be used to unlock rewards and achievements.
- **Challenges**: Create groups to challenge friends on who can achieve the most points in a given time frame. Participate in community challenges to stay motivated.
- **Progress Tracking**: Visualize your progress with graphs about your points and statistics over time. Track your weight loss and bioimpedance evolution to see how your body is changing and number of activities.
- **Social Interaction**: Connect with friends, share achievements, and encourage each other through the app's social features.
- **Rewards and Achievements**: Unlock badges and rewards for reaching milestones, completing challenges, and maintaining consistency in your fitness journey.

## Integrations
- **Strava**: Sync your Strava account to automatically import your activities and workouts.
- **Fitness Trackers**: Connect with popular fitness trackers to gather data on your workouts, weight, and bioimpedance metrics.
- **Health Apps**: Integrate with health apps to provide a comprehensive view of your fitness journey.

## Gamefication
FitFight incorporates gamefication elements to make fitness fun and engaging. Users can earn points, unlock achievements, and compete with friends in challenges. The app encourages users to stay active and motivated by providing a sense of accomplishment and friendly competition.
On each activity, users can earn points based on the intensity and duration of their workouts, as well as improvements in their weight and bioimpedance metrics. The app also features leaderboards to showcase top performers and foster a sense of community.
If a user loss weight without bioimpedance metrics, he win points based on the weight loss, but if he has bioimpedance metrics, he will win points based on the evolution of his bioimpedance metrics. If you lose weight but your bioimpedance shows that you lost only muscle mass, you will lose points too. But if you win muscle mass, you will win more points.
On each trophy on strava, the user will win points too. If a user make a new personal record, he will win points based on the improvement of his previous record. If a user make a new personal record on a segment, he will win points based on the improvement of his previous record on this segment. If a user make a new personal record on a route, he will win points based on the improvement of his previous record on this route. If a user make a new personal record on a route and on a segment, he will win points based on the improvement of his previous record on this route and on this segment. If a user make a new personal record on a route and on a segment and on an activity, he will win points based on the improvement of his previous record on this route and on this segment and on this activity.
If a user doesn't make any activity in a day, he will lose some points.
On registration the app must calculate the initial fitness point based on integrations data. The points inside a group should be relative. Every user must initialize a challenge with zero points, and each challenge has its growth points. But the user has it's own points that growth and decrease.
External activities (like running, cycling, swimming) must be best rewarded. A long external activity must be rewarded more than a short external activity. A long external activity must be rewarded more than a long internal activity.

If a user stay a week leading all his challenge, he will win a badge that is a picture made with ai. It must be his character with his physical characteristics, like his face, his body and his skin color. The character must be with his preferred sport clothes and with a medal on his chest. The character must be in a background that is the place of his preferred sport. 
If a user stay a month leading all his challenge, he will win a badge that is a picture made with ai. It must be his character with his physical characteristics, like his face, his body and his skin color. The character must be with his preferred sport clothes and with a medal on his chest. The character must be in a background that is the place of his preferred sport. The character must be in a podium with his friends applauding him.

If a user wins all his challenges, he will win a badge that is a picture made with ai. It must be his character with his physical characteristics, like his face, his body and his skin color. The character must be with his preferred sport clothes and with a medal on his chest. The character must be in a background that is the place of his preferred sport. The character must be in a throne with all his friends referring to him on their knees like subjects. The character must be with a crown on his head and a scepter in his hand.

It must be possible to share the badges on social media, and the app must have a gallery where users can see all their badges and achievements.

## User Experience
FitFight is designed to provide a seamless and enjoyable user experience. The app features an intuitive interface in mobile and web platforms.
It'll be allowed to log in via Google, Strava or create an account with email and password.

The home screen will display the user's current points, a line dashboard showing the gain and point losses over time, and cards about the user's challenge groups. Each card with a table showing the ranking with an arrow up or down based on the user's gain or loss of points compared to the previous day. The user can click on a card to see the details of the challenge, including the participants, their points, and the challenge's progress.

It must have a screen to create a challenge, adding registered participants and allowing to invite not registered participants through a link or by email. The user can set the challenge's duration.

It must have a screen to see the user's profile, with the user's integration and personal data like age, weight, height, and bioimpedance metrics. The profile screen will also display the user's achievements, badges, and rewards earned through the app.

It must have a screen to see the user's friends, with the option to add or remove friends, and view their profiles and achievements.

It must have a screen to see the user's activities, with the option to filter by date, type of activity, and points earned. The user can view detailed information about each activity, including distance, duration, calories burned, and points earned.

It must have a screen to see the user's statistics, with graphs and charts showing the user's progress over time, including points earned and lost and why, weight loss, and bioimpedance evolution. This screen must have an ia section that will analyze the user's data and provide insights and recommendations for improving their fitness journey.

The user search should use geolocation to find friends and other users nearby, allowing for local challenges and competitions. The app should also provide notifications for challenge updates, friend activities, and achievements. Even looking for name, should look nearby users first, and then in the whole database.

## Technical Requirements

The app should be developed using kotlin multiplatform for mobile and web platforms.

The backend should be a graalvm kotlin quarkus application with a PostgreSQL database.

The PostgreSQL database should be hosted in Digital Ocean, and the backend should be hosted in DigitalOcean app platform.

Everything should be managed via Terraform and Github.

It must be use Github Actions for CI/CD, with automated testing and deployment pipelines.

The PostgreSQL should use it's plugin to perform geolocation, queues and cache like redis.

To write the code consider a line of 180 characters, avoiding breaking lines on each parameter or work.

The backend should use folders for controllers, services, repositories, models, ai and utils. Each controller should have a corresponding service and repository. The models should be in a separate package.

The backend should use langchain4j and deepseek ai models.

The security must be done with JWT tokens, with refresh tokens and expiration times. The backend should have a rate limiter to prevent abuse and ensure fair usage.

The authentication and authorization should be done in a separate project using spring authorization server, so we can reuse in other projects.

## Monetization
FitFight must be free for users. But in the screen must appear randomly ads from personal trainers, groups that pay to be advertised, events and products. The ads must be relevant to the user's activities and interests, and should not interfere with the user experience. So when scrolling must appear some ads.

It must have a screen to look for professionals, events and products (the user must select a tab), with the option to filter by location, type, and price. The user can view detailed information about each professional, event, or product, including reviews and ratings from other users.

The app should also offer registration for companies and professionals to create their own profiles, advertise their services, and connect with potential clients. Companies and professionals can pay for premium features, such as enhanced visibility, analytics, and targeted advertising.

If it's an advertiser, the app should have a screen to manage their advertisements and payments.

The payments should be done via Stripe, with support for various payment methods, including credit cards, debit cards, and digital wallets. The app should also provide a secure and seamless checkout experience for users.



