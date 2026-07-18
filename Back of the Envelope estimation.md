Back of the Envelope estimation 

First calculate DAU : - 

Suppose your app has:

1 million users

Not all users are active every day.

Registered Users = 1,000,000

Daily Active Users (DAU) = 20%

= 200,000 users/day

Step 2: Estimate Requests Per User

Example:

Login → 1
View dashboard → 5
View loans → 10
Payments → 2
Logout → 1

Approximately:

20 requests/user/day

Total requests/day

200,000 × 20

= 4,000,000 requests/day

Step 3: Convert to Requests Per Second (RPS)
24 × 60 × 60

=

86,400 seconds/day

So

RPS

=

4,000,000 / 86,400

≈ 46 RPS


Traffic isn't evenly distributed.

Multiply by 3–5.

Peak RPS

=

50 × 5

=

250 RPS

Design for

250 RPS


Manage storage for the data 

bool or varchar - 2 bytes

decimal data - 8 bytes

images - 200 kB

Multiply them with number of user per day.


Step 6: Estimate Storage

Suppose

Loan record

2 KB

1 million loans

2 KB × 1,000,000

=

2 GB

Very manageable.

Documents?

Suppose each KYC PDF

500 KB

1 million users

500 KB × 1,000,000

=

500 GB


Step 7: Estimate Bandwidth

Suppose

Each API response

20 KB

Requests/day

4 million

Bandwidth/day

20 KB × 4,000,000

=

80 GB/day

Monthly

≈2.4 TB

Now CDN starts making sense for static content.

Step 8: Estimate Cache Size

Suppose

User object

5 KB

Active users

200,000

Cache

5 KB × 200,000

=

1 GB

A small Redis instance can easily handle this.

Step 9: Estimate Servers

Suppose one Spring Boot server handles

150 RPS

Peak traffic

250 RPS

Servers needed

250 / 150

≈2

Always keep spare capacity.

Deploy

3 servers

or u can count it using the number of request per server and number of request coming to the application.



Simply follow this steps 

How to Think During an Interview

Instead of memorizing formulas, follow this sequence:

Users
    ↓
Daily Active Users (DAU)
    ↓
Requests per user
    ↓
Requests/day
    ↓
Average RPS
    ↓
Peak RPS
    ↓
Reads vs Writes
    ↓
Storage
    ↓
Bandwidth
    ↓
Servers
    ↓
Database
    ↓
Cache
