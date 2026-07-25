# Ngân hàng câu hỏi phỏng vấn — Cloud Engineer (Middle)

Tổng cộng **70 câu hỏi** + **5 mẫu giới thiệu bản thân**.

---

## Mục lục

- [Giới thiệu bản thân](#giới-thiệu-bản-thân)
- [Dự án thực tế](#dự-án-thực-tế) — 18 câu
- [Kỹ thuật](#kỹ-thuật) — 35 câu
- [Hành vi & Kỹ năng mềm](#hành-vi-kỹ-năng-mềm) — 12 câu
- [Thiết kế hệ thống](#thiết-kế-hệ-thống) — 5 câu
- [Networking — Luồng đi & Concept](#networking-luồng-đi--concept) — 18 câu
- [Backup & Recovery (bổ sung)](#backup--recovery-bổ-sung) — 3 câu

---

## Giới thiệu bản thân

### SI01 — General / default opening

Tôi là Cloud Engineer tại FPT Software, khoảng 2.5 năm làm AWS và 5 năm backend với Java. Tôi có chứng chỉ AWS SAA. Phần lớn công việc là infrastructure as code với Terraform, cùng ECS, S3, IAM và CloudWatch. Tôi thích các task nằm giữa application và infrastructure, vì nền backend giúp tôi hiểu cả hai phía.

### SI02 — Security-oriented (for security-heavy roles)

Tôi là Cloud Engineer với 2.5 năm AWS. Nhiều công việc gần đây thiên về bảo mật: tinh chỉnh AWS WAF managed rules để giảm false positive, khóa S3 bằng Origin Access Control, và thiết kế IAM role cross-account. Trước khi làm cloud tôi có 5 năm viết Java, nên tôi hiểu ứng dụng thực sự cần gì trước khi siết quyền.

### SI03 — Data & migration oriented

Tôi là Cloud Engineer tập trung vào AWS, nền tảng backend Java. Dự án gần nhất là migrate database từ Aurora MySQL sang Aurora PostgreSQL. Chúng tôi dùng snapshot restore, transform dữ liệu qua S3 làm staging layer, rồi load bằng extension aws_s3. Tôi cũng làm việc với Athena để phân tích log và xây data pipeline trên ECS và Step Functions.

### SI04 — DevOps / automation oriented (your future direction)

Tôi là Cloud Engineer đang hướng sang DevOps. Tôi xây hạ tầng bằng Terraform và chạy CI/CD pipeline trên GitHub Actions, gồm cả self-hosted runner trong VPC để job truy cập được resource private. Tôi làm việc với Docker và ECS Fargate hàng ngày. Hiện tôi đang học sâu hơn về Linux và networking, vì tôi muốn vững nền tảng chứ không chỉ biết dùng AWS console.

### SI05 — Growth / learning oriented (good for culture-fit rounds)

Tôi là Cloud Engineer tại FPT Software, khoảng 2.5 năm AWS sau 5 năm backend Java. Điều tôi thích nhất là troubleshooting — ví dụ query Athena đột nhiên trả 0 rows và tôi lần ra nguyên nhân là log format thay đổi. Ngoài giờ làm tôi học Linux, Docker và networking. Tôi muốn phát triển lên vai trò DevOps nên đang xây nền tảng từ bây giờ.

---

## Dự án thực tế

### Q001 — Trong task WAF, bạn chạy managed rules ở count mode trước. Tại sao không chuyển thẳng sang block mode?

> `waf` · độ khó 2/3

Chúng tôi dùng AWS Managed Rules nhưng không biết API nào sẽ bị match. Một số endpoint gửi request body trên 8KB nên bị SizeRestrictions_BODY bắt. Bật block mode ngay sẽ gây false positive trên production. Vì vậy chúng tôi dùng count mode trước. Rule vẫn gắn label nhưng không chặn. Sau đó tôi kiểm tra WAF logs trên Athena để xem URI nào match.

### Q002 — Giải thích cơ chế label hoạt động thế nào trong setup WAF của bạn?

> `waf` · độ khó 3/3

Đầu tiên tôi dùng rule_action_override để đổi một sub-rule sang count. Rule vẫn chạy và vẫn gắn label, chỉ là không chặn. Label đó theo request. Sau đó tôi thêm custom rule phía sau. Rule này match theo label nhưng loại trừ các URI được whitelist. Nên mọi thứ có label sẽ bị chặn, trừ API của chúng tôi. Thứ tự rule rất quan trọng.

### Q003 — Query Athena đột nhiên trả về 0 rows. Kể tôi nghe bạn tìm ra nguyên nhân gốc thế nào?

> `athena` · độ khó 3/3

Query trả 0 rows nhưng file log vẫn đổ về S3. Nghĩa là ingestion ổn, vấn đề ở parsing. Tôi tải một file cũ và một file mới rồi so sánh. AWS đã thêm field mới vào cuối mỗi dòng. Bảng của chúng tôi dùng RegexSerDe vốn cần số cột cố định. Khi dòng không match, Athena âm thầm bỏ record. Tôi thêm nhóm catch-all vào cuối regex.

### Q004 — Bạn sẽ phòng ngừa vấn đề log format đó tái diễn thế nào?

> `athena` · độ khó 2/3

Ba điều. Thứ nhất, viết regex theo hướng khoan dung, có catch-all ở cuối. Thứ hai, thêm CloudWatch alarm khi số dòng mỗi ngày về 0, để phát hiện trong vài giờ chứ không phải vài tuần. Thứ ba, nếu format log cho phép thì chuyển sang SerDe hiểu schema như JSON thay vì regex, vì nó không phụ thuộc thứ tự cột.

### Q005 — Tại sao bạn export dữ liệu ra S3 dạng CSV thay vì nối trực tiếp MySQL với PostgreSQL?

> `migration` · độ khó 2/3

Hai engine khác nhau, và schema đã đổi do re-architecture. Nên đây không phải copy. Chúng tôi phải map entity và làm sạch dữ liệu. Chuyển trực tiếp thì tôi không kiểm tra được kết quả. Dùng S3 làm staging layer giúp tách rời export và import. Nếu import lỗi, tôi chạy lại mà không phải động vào nguồn. Sau đó tôi load bằng extension aws_s3.

### Q006 — Tại sao bạn tạo index sau khi load dữ liệu thay vì trước?

> `migration` · độ khó 2/3

Nếu bảng đã có index, mỗi dòng insert đều phải cập nhật index. Với bulk load thì rất chậm. Nếu load trước rồi tạo index một lần ở cuối, database làm được trong một lượt. Sau đó tôi chạy ANALYZE để làm mới statistics, vì query planner cần statistics mới để chọn plan tốt. Rồi tôi kiểm tra hiệu năng query.

### Q007 — Bạn xác minh migration đúng bằng cách nào?

> `migration` · độ khó 3/3

Nhiều tầng. Thứ nhất, so sánh row count từng bảng giữa nguồn và đích. Thứ hai, tôi kiểm tra mẫu các bảng nghiệp vụ quan trọng. Thứ ba, tôi xem việc chuyển đổi kiểu dữ liệu, vì MySQL và PostgreSQL xử lý boolean và date khác nhau. Thứ tư, chúng tôi chạy query thật của ứng dụng và so sánh kết quả. Cuối cùng tôi kiểm tra hiệu năng query sau khi tạo index.

### Q008 — Tại sao dùng Step Functions giữa EventBridge và ECS thay vì để EventBridge chạy task trực tiếp?

> `batch` · độ khó 2/3

EventBridge có thể chạy ECS task, nhưng về cơ bản là fire-and-forget — nó không chờ và không biết task lỗi hay không. Step Functions theo dõi trạng thái task đến khi xong, và có retry sẵn kèm backoff. Nó cũng cho execution history, giúp debug dễ hơn nhiều. Nếu sau này cần thêm bước, chúng tôi thêm được mà không phải viết lại trigger.

### Q009 — Giải thích cơ chế upload S3 cross-account trong hệ thống batch của bạn.

> `batch` · độ khó 3/3

ECS task chạy với task role trong account của chúng tôi. Ở account đích có role thứ hai tin cậy account chúng tôi. Code Java gọi sts:AssumeRole lên role đó và nhận temporary credentials. Nó dùng credentials này để upload file. Chi tiết quan trọng là object ownership — nếu cấu hình sai, chủ bucket không đọc được file vừa upload.

### Q010 — Tại sao bạn để credentials database trong SSM Parameter Store thay vì environment variable?

> `batch` · độ khó 2/3

Environment variable được lưu trong task definition, nên ai có quyền console đều đọc được, và thường lọt vào Terraform state hoặc git. Với SSM Parameter Store, giá trị được lấy lúc runtime và mã hóa được bằng KMS. Chúng tôi có thể xoay vòng secret mà không cần deploy lại container. Quyền truy cập do IAM kiểm soát nên tôi giới hạn được role nào đọc parameter nào.

### Q011 — Bạn dùng CloudFront Functions thay vì Lambda@Edge. Điều gì khiến đó là lựa chọn đúng?

> `edge` · độ khó 3/3

Logic của chúng tôi rất nhẹ — một redirect, một HTTP 426, và một JSON nhỏ. CloudFront Functions chạy ngay tại edge location, dưới một mili-giây, và rẻ hơn nhiều. Lambda@Edge chạy ở regional edge cache nên thêm độ trễ. Hạn chế là CloudFront Functions không gọi được network. Chúng tôi không cần nên hạn chế đó không ảnh hưởng.

### Q012 — Origin Access Control bảo vệ S3 bucket của bạn thế nào?

> `edge` · độ khó 2/3

Không có nó, người ta có thể vào thẳng URL S3 và bỏ qua CloudFront, nghĩa là bỏ qua luôn WAF và cache. Với Origin Access Control, CloudFront ký request gửi tới S3. Bucket policy khi đó chỉ cho phép request từ đúng distribution đó, dùng điều kiện aws:SourceArn. Nên bucket ở chế độ private và mọi traffic buộc phải đi qua CloudFront.

### Q013 — Tại sao upload file vào staging prefix thay vì thẳng vào prefix của Adobe Analytics?

> `transfer-family` · độ khó 2/3

Nếu người dùng upload thẳng vào prefix thật, file lỗi sẽ đi thẳng vào quy trình import và chúng tôi không chặn được. Với staging prefix, workflow của Transfer Family chạy sau khi upload xong. Một Lambda kiểm tra kích thước file, Lambda khác đổi tên theo naming convention và di chuyển file. Nên chỉ file đã được kiểm mới tới prefix mà Adobe Analytics đọc.

### Q014 — Home directory mapping trong AWS Transfer Family hoạt động thế nào và tại sao bạn dùng nó?

> `transfer-family` · độ khó 2/3

Mỗi user được map tới prefix S3 riêng. Phía SFTP họ chỉ thấy thư mục gốc là dấu gạch chéo. Họ không thấy đường dẫn thật của bucket và không duyệt được thư mục của khách khác. Cách này cho cô lập theo tenant mà không cần tạo bucket riêng cho từng khách. Nó cũng giữ trải nghiệm y như FTP cũ, điều này quan trọng vì người dùng vẫn dùng WinSCP.

### Q015 — Bạn đã có RDS automated backup rồi. Tại sao còn thêm AWS Backup?

> `backup` · độ khó 2/3

RDS point-in-time recovery chỉ tối đa 35 ngày. Cái đó lo được lỗi vận hành, nhưng yêu cầu compliance là lưu dài hạn 5 năm. AWS Backup cho phép định nghĩa hai rule — daily giữ 30 ngày, monthly giữ 5 năm — trong một vault tập trung. Nó cũng cho một chỗ duy nhất để audit toàn bộ backup thay vì kiểm từng database riêng.

### Q016 — Bạn dùng Backup Vault Lock ở Governance mode. Nó làm gì, và tại sao không dùng Compliance mode?

> `backup` · độ khó 3/3

Vault Lock ngăn việc xóa backup sớm, kể cả xóa nhầm. Ở Governance mode, admin có quyền đặc biệt vẫn override được. Ở Compliance mode, không ai đổi được — kể cả root account — cho tới khi lock hết hạn. Chúng tôi mới vận hành 6 tháng đầu nên cấu hình còn thay đổi. Compliance mode không thể đảo ngược nên quá rủi ro ở giai đoạn đó.

### Q017 — Giải thích thiết kế log retention và tại sao export CloudWatch Logs sang S3?

> `backup` · độ khó 2/3

Chúng tôi chỉ giữ 30 ngày trong CloudWatch vì lưu trữ CloudWatch đắt cho dữ liệu dài hạn. Mỗi ngày EventBridge kích hoạt một Lambda gọi CreateExportTask để đẩy log sang S3. Trên S3 chúng tôi dùng lifecycle policy: Standard 30 ngày, rồi Standard-IA 90 ngày, rồi Glacier cho phần còn lại, tổng cộng một năm. Nên vẫn giữ được log mà trả ít tiền hơn nhiều.

### Q018 — Mô tả chiến lược rollback khi deployment thất bại.

> `backup` · độ khó 2/3

Hai tầng. Với ứng dụng, ECR giữ ba image gần nhất nên chúng tôi deploy lại phiên bản trước rất nhanh. Với dữ liệu, pipeline tạo RDS snapshot trước mỗi lần release và giữ ba bản gần nhất. Rollback code thì nhanh. Rollback dữ liệu mới là phần khó, vì restore snapshot mất thời gian và mất hết dữ liệu ghi sau đó. Nên đó là phương án cuối cùng.

---

## Kỹ thuật

### Q019 — Bạn thêm aws:SourceAccount vào bucket policy cho CloudWatch Logs export. Điều kiện đó ngăn vấn đề gì?

> `iam` · độ khó 2/3

Policy cấp quyền cho service principal logs.<region>.amazonaws.com. Principal này không gắn với account nào cụ thể. Nếu không có condition, CloudWatch Logs ở bất kỳ account nào cũng export được vào bucket của tôi. Đó là confused deputy problem — một service đáng tin bị lợi dụng để hành động thay kẻ khác. aws:SourceAccount thu hẹp quyền xuống đúng một account ID.

### Q020 — Khác biệt giữa identity-based policy và resource-based policy là gì?

> `iam` · độ khó 2/3

Identity-based policy gắn vào user hoặc role và nói identity đó làm được gì. Resource-based policy gắn vào chính resource và nói ai được truy cập nó. Chỉ resource-based mới có trường Principal. Với truy cập cross-account, thường cả hai phía đều phải cho phép. S3 bucket policy là ví dụ phổ biến nhất.

### Q021 — IAM đánh giá một request thế nào khi có nhiều policy?

> `iam` · độ khó 3/3

Mặc định mọi thứ là implicit deny. IAM kiểm tra explicit Deny trước — nếu tìm thấy ở bất kỳ đâu thì request bị từ chối và không gì ghi đè được. Sau đó nó tìm Allow. Nếu không có Allow, request vẫn bị từ chối. Ngoài ra, service control policy và permission boundary chỉ giới hạn được chứ không bao giờ cấp quyền.

### Q022 — Tại sao EC2 nên dùng IAM role thay vì access key?

> `iam` · độ khó 2/3

Access key là loại dài hạn và phải lưu ở đâu đó — file config, biến môi trường, hoặc tệ hơn là trong git. Role cấp cho instance temporary credentials qua instance metadata service, và chúng tự xoay vòng. Nếu cần thu hồi quyền, tôi đổi role và có hiệu lực ngay, không phải deploy lại và không có key nào phải dọn.

### Q023 — Object Ownership đặt là BucketOwnerEnforced thực chất làm gì?

> `s3` · độ khó 2/3

Nó vô hiệu hóa hoàn toàn ACL. Chủ bucket tự động sở hữu mọi object, kể cả object do account khác ghi vào. Điều này tránh vấn đề kinh điển: account A upload file mà account B là chủ bucket lại không đọc được object của chính mình. Sau đó quyền truy cập chỉ do policy kiểm soát, đơn giản hơn và là mặc định AWS khuyến nghị.

### Q024 — Kể các S3 storage class bạn dùng cho log trong một năm.

> `s3` · độ khó 2/3

Standard cho 30 ngày đầu vì đó là lúc chúng tôi thực sự query log. Sau đó Standard-IA, lưu rẻ hơn nhưng tính phí khi lấy ra. Rồi Glacier cho phần còn lại của năm, rẻ nhất nhưng restore mất thời gian. Lifecycle policy tự động chuyển object. Một điểm cần lưu ý là thời gian lưu tối thiểu ở các class lạnh.

### Q025 — Khi nào bạn chọn ECS Fargate thay vì EC2 launch type?

> `ecs` · độ khó 2/3

Fargate nghĩa là không có server phải vá lỗi và không phải quản lý capacity cụm. Điều đó hợp với batch chạy mỗi ngày một lần, vì dùng EC2 thì phải trả tiền cho instance nhàn rỗi. EC2 launch type tốt hơn khi mức dùng ổn định và cao, vì rẻ hơn ở quy mô lớn, hoặc khi cần kiểm soát nhiều hơn — GPU, kernel riêng, hay daemon container.

### Q026 — Khác biệt giữa ECS task role và task execution role là gì?

> `ecs` · độ khó 3/3

Execution role được ECS dùng, trước khi container khởi động — để pull image từ ECR và tạo log stream trong CloudWatch. Task role được ứng dụng bên trong container dùng, khi nó gọi S3 hay SSM. Nếu nhầm hai cái, bạn sẽ gặp lỗi không pull được image lúc khởi động, hoặc access denied lúc chạy.

### Q027 — Retry trong Step Functions hoạt động thế nào, và khi nào retry là ý tồi?

> `stepfunctions` · độ khó 2/3

Bạn định nghĩa khối Retry với loại lỗi, khoảng chờ, số lần tối đa và hệ số backoff. Nó hợp với lỗi tạm thời như throttling hay timeout mạng. Nó dở với lỗi vĩnh viễn — config sai hay dữ liệu không hợp lệ sẽ lỗi y hệt mọi lần, chỉ tốn thời gian. Nó cũng nguy hiểm nếu job không idempotent, vì retry có thể tạo dữ liệu trùng.

### Q028 — Khác biệt giữa Aurora writer endpoint và reader endpoint là gì?

> `aurora` · độ khó 2/3

Writer endpoint luôn trỏ tới instance primary hiện tại. Nếu xảy ra failover, nó tự động theo primary mới nên ứng dụng không phải đổi gì. Reader endpoint phân tải đọc qua các replica. Điều cần cẩn thận là replication lag — nếu bạn ghi rồi đọc ngay từ reader endpoint, có thể không thấy dữ liệu vừa ghi.

### Q029 — Làm sao giảm chi phí một query Athena?

> `athena` · độ khó 2/3

Athena tính tiền theo lượng dữ liệu quét, nên mục tiêu là quét ít đi. Thứ nhất, phân vùng bảng theo ngày để query chỉ đọc prefix cần thiết. Thứ hai, dùng định dạng cột như Parquet để chỉ đọc những cột bạn chọn. Thứ ba, nén file. Và tránh SELECT * — chọn hết cột thì mất luôn lợi ích của định dạng cột.

### Q030 — Cache của CloudFront hoạt động thế nào và invalidate nó ra sao?

> `cloudfront` · độ khó 2/3

CloudFront lưu response tại edge, đánh dấu bằng cache key — đường dẫn cộng với header hoặc query string mà bạn chọn đưa vào. TTL lấy từ cache policy hoặc header của origin. Để buộc làm mới, bạn tạo invalidation, nhưng làm thường xuyên thì tốn tiền. Cách tốt hơn là đặt tên file có phiên bản, khi đó bản mới đơn giản là một cache key mới.

### Q031 — Cold start của Lambda là gì và giảm nó thế nào?

> `lambda` · độ khó 2/3

Cold start xảy ra khi Lambda phải tạo môi trường chạy mới — tải code, khởi động runtime, chạy phần khởi tạo. Nó tệ hơn với package lớn hoặc runtime nặng như JVM. Để giảm: giữ package nhỏ, đưa phần setup như database client ra ngoài handler để tái sử dụng, và nếu độ trễ thực sự quan trọng thì dùng provisioned concurrency, dù nó tốn tiền.

### Q032 — Bạn sẽ thiết lập cảnh báo cho batch phải xong trước 6 giờ sáng thế nào?

> `monitoring` · độ khó 2/3

Thứ nhất, CloudWatch alarm trên metric số execution thất bại của Step Functions, gửi qua SNS rồi tới Slack. Thứ hai, alarm trên thời gian chạy, để lần chạy chậm cảnh báo trước hạn. Trường hợp khó hơn là job không chạy lần nào — alarm lỗi sẽ không kêu vì có gì lỗi đâu. Cho tình huống đó tôi thêm một kiểm tra lúc 6 giờ sáng, báo động nếu không có lần chạy thành công nào.

### Q033 — Mã hóa KMS cho một object S3 hoạt động thế nào, nói đơn giản?

> `kms` · độ khó 2/3

Master key nằm trong KMS và không bao giờ ra khỏi đó. Khi upload, S3 xin KMS một data key, mã hóa object bằng key đó, và lưu data key đã mã hóa cạnh object. Để đọc object, S3 nhờ KMS giải mã data key đó. Kết quả thực tế là người đọc cần hai quyền — quyền đọc S3 và quyền decrypt của KMS.

### Q034 — Ai đó sửa tay resource trên console. Lần terraform apply tiếp theo sẽ ra sao?

> `terraform` · độ khó 3/3

Terraform làm mới state và thấy sự khác biệt. Plan sẽ hiện rằng nó muốn đưa thay đổi thủ công về đúng như trong code. Sự khác biệt đó gọi là configuration drift. Cách sửa đúng không phải cãi với Terraform mà là đưa thay đổi vào code. Để phòng ngừa, chúng tôi hạn chế quyền ghi trên console và để CI là con đường duy nhất chạy apply.

### Q035 — Tại sao Terraform state cần lưu từ xa và có locking?

> `terraform` · độ khó 2/3

File state ánh xạ code sang resource thật. Nếu nó chỉ nằm trên máy tôi thì không ai khác apply được, và mất file là mất dấu mọi thứ. Remote backend như S3 chia sẻ nó cho cả team và CI. State locking quan trọng vì nếu hai người apply cùng lúc, state có thể hỏng và resource bị tạo trùng hoặc bị xóa.

### Q036 — Khác biệt giữa EventBridge Scheduler và EventBridge rule là gì?

> `eventbridge` · độ khó 2/3

EventBridge rule nằm trên event bus và phản ứng với event — ví dụ ECS task đổi trạng thái. Nó cũng chạy theo lịch được nhưng đó là tính năng phụ. EventBridge Scheduler được thiết kế riêng cho lịch chạy. Nó hỗ trợ múi giờ, lịch chạy một lần, khung thời gian linh hoạt và cấu hình retry riêng. Với batch hằng ngày, Scheduler hợp hơn.

### Q037 — AWS Config dùng để làm gì và khác CloudTrail thế nào?

> `awsconfig` · độ khó 2/3

CloudTrail ghi lại ai làm gì — là lịch sử lệnh gọi API. AWS Config ghi lại resource trông như thế nào theo thời gian và đánh giá theo các rule tuân thủ. Nên CloudTrail trả lời ai đã xóa cấu hình, còn Config trả lời cấu hình hiện tại có sai không. Ở dự án của tôi, Config theo dõi cấu hình backup và retention, cảnh báo đi qua SNS tới Slack.

### Q038 — Khác biệt giữa security group và network ACL là gì?

> `networking` · độ khó 2/3

Chúng làm việc ở tầng khác nhau. Security group gắn vào network interface nên bảo vệ một resource. Network ACL làm việc ở tầng subnet. Khác biệt then chốt: security group là stateful — cho traffic vào thì chiều trả về tự động ra được. NACL là stateless nên phải viết cả hai chiều. NACL cũng deny tường minh được, và rule xét theo số, khớp đầu tiên thắng.

### Q039 — Tại sao NACL cần rule cho ephemeral port còn security group thì không?

> `networking` · độ khó 3/3

Khi client mở kết nối, nó chọn một cổng cao ngẫu nhiên làm cổng nguồn, thường trên 1024. Server trả lời về đúng cổng đó. Security group là stateful nên nhớ kết nối và tự cho gói trả về đi qua. NACL là stateless — mỗi gói được xét riêng. Nên gói trả về trông như traffic mới, và bạn phải cho phép dải ephemeral port ở chiều ra một cách tường minh.

### Q040 — Resource trong private subnet ra internet bằng cách nào?

> `networking` · độ khó 2/3

Private subnet không có route tới internet gateway nên không ra thẳng được. Chúng tôi đặt NAT Gateway trong public subnet, và route table của private gửi 0.0.0.0/0 tới NAT Gateway đó. NAT chỉ cho chiều ra — không ai từ internet mở kết nối vào được. Đánh đổi là chi phí, vì NAT tính tiền theo giờ và theo mỗi gigabyte xử lý.

### Q041 — VPC endpoint là gì và khi nào bạn dùng nó?

> `networking` · độ khó 3/3

Nó cho phép resource trong private subnet tới các dịch vụ AWS mà không phải đi qua internet. Có hai loại. Gateway endpoint dùng cho S3 và DynamoDB, miễn phí, hoạt động qua route table. Interface endpoint dùng cho hầu hết dịch vụ khác, dùng IP nội bộ qua ENI, tính tiền theo giờ. Lợi ích chính là tránh phí dữ liệu qua NAT và giữ traffic trong mạng AWS.

### Q042 — Bạn quyết định kích thước CIDR khi thiết kế VPC thế nào?

> `networking` · độ khó 2/3

Nguyên tắc chính là dự trù lớn hơn nhu cầu, vì sau này khó thu nhỏ. Thứ hai, tránh CIDR chồng lấn với on-premises hay VPC khác, nếu không peering sẽ không chạy. Sau đó tôi chia theo availability zone, mỗi zone có subnet public và private. Hai chi tiết hay bị quên: AWS giữ lại năm địa chỉ IP trong mỗi subnet, và mỗi Fargate task chiếm một IP.

### Q043 — Tại sao cần subnet ở ít nhất hai availability zone?

> `networking` · độ khó 2/3

Một availability zone là điểm lỗi đơn — nếu trung tâm dữ liệu đó gặp sự cố thì cả hệ thống chết. AWS cũng bắt buộc về mặt cấu trúc: load balancer và RDS multi-AZ đều cần subnet ở ít nhất hai zone, vì failover phải có nơi để chuyển sang. Điều cần nhớ là truyền dữ liệu giữa các AZ bị tính phí.

### Q044 — EC2 trong private subnet không kết nối được RDS. Bạn troubleshoot thế nào?

> `networking` · độ khó 3/3

Tôi đi từng tầng một. Đầu tiên là route table — có route giữa hai subnet không. Thứ hai là security group của database: nó có cho phép chiều vào ở cổng database từ security group của EC2 không, chứ không phải từ một IP. Thứ ba là NACL, ở cả hai chiều, vì nó stateless. Thứ tư, endpoint có phân giải đúng không. Xong hết mới xét tới thông tin đăng nhập.

### Q045 — Khác biệt giữa public subnet và private subnet là gì?

> `networking` · độ khó 2/3

Khác biệt không nằm ở một ô tích trên subnet mà ở route table. Public subnet có route 0.0.0.0/0 trỏ tới internet gateway. Private subnet thì không; hoặc không có route ra internet, hoặc trỏ tới NAT Gateway. Ngoài ra, resource trong public subnet vẫn cần IP public mới truy cập được từ ngoài. Và private không tự nhiên có nghĩa là an toàn.

### Q046 — Tại sao team bạn chạy GitHub Actions trên self-hosted runner thay vì runner của GitHub?

> `cicd` · độ khó 2/3

Job migration phải kết nối tới Aurora trong private subnet. Runner của GitHub nằm ngoài mạng nội bộ nên không tới được. Phương án thay thế là mở database ra public, điều chúng tôi không làm. Nên chúng tôi chạy runner trên EC2 bên trong VPC. Nó cũng dùng được instance role thay vì access key dài hạn. Đánh đổi là chúng tôi phải tự vá lỗi và nó không tự scale.

### Q047 — Self-hosted runner có rủi ro bảo mật gì?

> `cicd` · độ khó 3/3

Runner nằm trong mạng nội bộ và có quyền thật. Nếu ai đó mở pull request chứa code độc, code đó chạy trên máy chúng tôi. Nên chúng tôi không bao giờ bật nó cho repository public hay pull request từ fork. Rủi ro khác là workspace không được dọn giữa các job, nên file hay credentials có thể rò từ job này sang job khác. Chúng tôi giảm rủi ro bằng runner dùng một lần và IAM role tối thiểu.

### Q048 — Bạn giảm kích thước Docker image bằng cách nào?

> `docker` · độ khó 2/3

Cải thiện lớn nhất là multi-stage build — biên dịch ở stage đầu, rồi chỉ copy artifact sang image cuối sạch sẽ, nên công cụ build không đi kèm. Thứ hai, dùng base image nhỏ hơn, ví dụ JRE slim thay vì JDK đầy đủ. Thứ ba, gộp các lệnh RUN vì mỗi lệnh tạo một layer. Và dùng .dockerignore. Image nhỏ thì pull nhanh hơn, quan trọng với thời gian khởi động Fargate.

### Q049 — Tại sao dùng tag 'latest' là thói quen xấu?

> `docker` · độ khó 2/3

latest là tag có thể thay đổi — theo thời gian nó trỏ tới nội dung khác nhau. Nên bạn không biết production đang chạy phiên bản nào, và rollback thành bất khả thi vì bản latest trước đó đã mất. Hai môi trường cũng có thể chạy code khác nhau với cùng một tag. Tôi thích tag bất biến, ví dụ commit SHA, để một tag luôn ứng với đúng một bản build.

### Q050 — Server hết dung lượng ổ đĩa. Bạn tìm thứ đang chiếm chỗ thế nào?

> `linux` · độ khó 2/3

Tôi bắt đầu bằng df -h để xem filesystem nào đầy — thường không phải cái mọi người tưởng. Rồi du -sh /* và đào dần vào thư mục lớn nhất. Thường là /var/log hoặc các layer của Docker. Một trường hợp hay gây bối rối: file đã bị xóa nhưng một tiến trình vẫn giữ nó mở nên dung lượng chưa được trả lại. Lệnh lsof cho thấy điều đó. Sau đó tôi xử lý gốc bằng log rotation.

### Q051 — Trên Linux bạn kiểm tra cái gì đang lắng nghe trên một cổng và service có khỏe không?

> `linux` · độ khó 2/3

Tôi dùng ss -tulpn để liệt kê các cổng đang lắng nghe kèm tên tiến trình. Rồi systemctl status để xem trạng thái service và journalctl -u để xem log. Tôi cũng curl localhost để thử ngay trên chính máy đó. Bước cuối rất quan trọng: nếu chạy được cục bộ mà từ ngoài không được, thì ứng dụng ổn và vấn đề nằm ở tầng mạng — security group, NACL, hoặc route.

### Q052 — Quyền file Linux như 644 và 600 nghĩa là gì, và tại sao SSH quan tâm?

> `linux` · độ khó 2/3

Ba chữ số ứng với chủ sở hữu, nhóm, và những người khác. Bốn là đọc, hai là ghi, một là thực thi. Nên 644 là chủ đọc ghi, người khác chỉ đọc. 600 là chỉ chủ sở hữu. SSH quan tâm vì private key không được để người khác đọc — nếu để, SSH từ chối dùng và kết nối thất bại. Vì thế file key cần quyền 600.

### Q053 — Bạn xử lý secret trong CI/CD pipeline thế nào?

> `cicd` · độ khó 2/3

Không bao giờ hardcode và không bao giờ commit. Tôi lưu chúng trong secret store của CI, hoặc tốt hơn là lấy lúc runtime từ SSM hay Secrets Manager. Với quyền AWS, tôi thích OIDC hơn để pipeline nhận role tạm thời thay vì key tĩnh. Hệ thống CI có che secret trong log, nhưng chỉ khớp chính xác mới che được, nên giá trị mã hóa base64 vẫn có thể lộ. Và phải xoay vòng định kỳ.

---

## Hành vi & Kỹ năng mềm

### Q054 — Kể về lần hai team có yêu cầu mâu thuẫn nhau. Bạn xử lý thế nào?

> `conflict` · độ khó 2/3

Bên bảo mật muốn bật AWS Managed Rules, nhưng nhiều API của chúng tôi gửi request body lớn và bị rule kích thước bắt. Thay vì chọn phe, tôi đề nghị đo trước. Chúng tôi chạy count mode và tôi phân tích log trên Athena, nhờ vậy có dữ liệu thay vì ý kiến. Sau đó tôi đề xuất chỉ override những sub-rule đó và chặn theo label kèm whitelist. Cả hai bên đều đạt được điều họ cần.

### Q055 — Kể về lần có sự cố trên production và bạn phải sửa dưới áp lực.

> `pressure` · độ khó 2/3

Query Athena đột nhiên trả 0 rows nên chúng tôi mất khả năng quan sát log WAF và CloudFront. Đầu tiên tôi xác nhận dữ liệu vẫn về S3, điều đó cho thấy vấn đề ở parsing chứ không phải ingestion. Sau đó tôi so file log cũ với file mới và thấy AWS đã thêm field mới. Tôi sửa regex bằng nhóm catch-all để vẫn tương thích ngược. Bài học chính là nó lỗi trong im lặng.

### Q056 — Kể về lần bạn phải giải thích điều kỹ thuật cho người không rành kỹ thuật.

> `communication` · độ khó 2/3

Khi thay FTP server bằng AWS Transfer Family, khách hàng phải chuyển từ mật khẩu sang SSH key pair. Họ là người dùng nghiệp vụ. Nên tôi tránh hẳn thuật ngữ và dùng ẩn dụ: public key là ổ khóa bạn đưa để tôi lắp, private key là chìa bạn giữ. Rồi một quy tắc duy nhất cần nhớ — đừng gửi private key cho tôi. Tôi cũng viết hướng dẫn từng bước kèm ảnh chụp màn hình.

### Q057 — Kể về một sai lầm bạn từng mắc và bạn học được gì.

> `mistake` · độ khó 2/3

Thời gian đầu tôi sửa trực tiếp trên console khi có việc gấp, thay vì đi qua Terraform. Lúc đó chạy được, nhưng sau đó pipeline hoàn nguyên thay đổi của tôi vì code không biết gì về nó. Đó là configuration drift. Không ai bị ảnh hưởng, nhưng tôi học được là phải đưa thay đổi vào code trước, kể cả khi chậm hơn. Giờ nguyên tắc của tôi là: không có trong code thì coi như không tồn tại.

### Q058 — Bạn học công nghệ mới thế nào?

> `learning` · độ khó 2/3

Tôi bắt đầu từ tài liệu chính thức, vì bài blog thường lỗi thời. Sau đó tôi làm một thứ nhỏ nhưng thật, vì tôi chỉ nhớ những gì đã thực sự dùng. Tôi cũng cố tình làm hỏng nó, vì hiểu các kiểu lỗi dạy được nhiều hơn đường đi suôn sẻ. Hiện tại tôi đang học Linux, Docker và networking. Tôi muốn hiểu cái nằm dưới AWS console chứ không chỉ biết bấm nút nào.

### Q059 — Tại sao bạn muốn chuyển từ Cloud Engineer sang DevOps?

> `career` · độ khó 2/3

Tôi đã làm một phần rồi — Terraform, GitHub Actions, deploy container. Và 5 năm Java giúp tôi hiểu lập trình viên cần gì chứ không chỉ hạ tầng cho phép gì. Tôi muốn làm chủ toàn bộ con đường đưa sản phẩm ra, từ commit tới production, thay vì chỉ phần hạ tầng. Vì thế tôi đang xây nền tảng: Linux, Docker và networking. Về lâu dài tôi quan tâm tự động hóa và độ tin cậy.

### Q060 — Kể về lần bạn không đồng ý với quyết định thiết kế của người cấp cao hơn.

> `disagreement` · độ khó 2/3

Bước đầu của tôi là đặt câu hỏi, vì thường có ràng buộc mà tôi chưa biết. Nếu vẫn không đồng ý, tôi đưa ra một lo ngại cụ thể kèm bằng chứng chứ không phải sở thích cá nhân — ví dụ chỉ ra điều gì xảy ra khi rollback. Trong thiết kế backup, tôi đã nêu lo ngại về việc thực sự khôi phục được tới đâu. Nếu quyết định vẫn theo hướng khác, tôi chấp nhận và thực hiện, nhưng ghi lại rủi ro.

### Q061 — Kể về lần bạn cải thiện thứ gì đó mà không ai yêu cầu.

> `ownership` · độ khó 2/3

Sau sự cố Athena, không ai yêu cầu tôi làm gì thêm ngoài việc sửa lỗi. Nhưng vấn đề thật là nó lỗi trong im lặng một thời gian trước khi ai đó phát hiện. Nên tôi đề xuất thêm cảnh báo trên số dòng mỗi ngày, để việc tụt về 0 được phát hiện nhanh. Đó là thay đổi nhỏ, nhưng biến lỗi im lặng thành lỗi nhìn thấy được. Tôi cố tìm những chỗ như vậy — sửa thì rẻ mà chi phí lặp lại thì cao.

### Q062 — Bạn làm việc với người khác múi giờ hoặc khác ngôn ngữ thế nào?

> `teamwork` · độ khó 2/3

Tôi dựa vào giao tiếp bằng văn bản, vì cuộc họp chỉ giúp những người tham dự. Tôi viết tóm tắt ngắn gồm quyết định và lý do, để ai cũng theo kịp một cách bất đồng bộ. Khi ngôn ngữ là rào cản, tôi xác nhận lại cách hiểu chứ không đoán — tôi nhắc lại điều tôi nghĩ đã thống nhất. Tôi cũng dùng sơ đồ, vì hình vẽ loại bỏ nhiều mơ hồ mà chữ nghĩa gây ra.

### Q063 — Bạn ưu tiên thế nào khi có nhiều việc gấp cùng lúc?

> `priority` · độ khó 2/3

Đầu tiên tôi hỏi có gì đang ảnh hưởng production hay khách hàng không, vì cái đó luôn đứng đầu. Tiếp theo, việc nào đang chặn người khác — nếu việc của tôi chặn hai người thì đáng giá hơn việc chỉ chặn mình tôi. Sau đó tôi cân giữa hạn chót và công sức. Phần quan trọng nhất là báo sớm nếu có gì sẽ trễ. Trễ hạn trong im lặng tệ hơn nhiều so với nói trước.

### Q064 — Bạn có câu hỏi nào cho chúng tôi không?

> `questions` · độ khó 2/3

Vâng, tôi có vài câu. Thứ nhất, team hiện xử lý on-call và sự cố thế nào? Thứ hai, deployment pipeline hiện tại ra sao, và điều anh chị muốn cải thiện nhất ở nó là gì? Thứ ba, thành công của vị trí này trong sáu tháng đầu trông như thế nào? Tôi hỏi câu cuối vì tôi muốn biết vấn đề thực sự mà công ty đang tuyển người để giải quyết.

### Q065 — Điểm yếu lớn nhất của bạn là gì?

> `strength-weakness` · độ khó 2/3

Nói tiếng Anh. Đọc và viết thì ổn vì tôi làm việc với tài liệu hằng ngày, nhưng nói thì chậm hơn, nhất là với từ vựng chuyên ngành. Tôi đang chủ động cải thiện — tôi luyện mỗi ngày và tập trung vào những thuật ngữ thực sự dùng trong công việc. Tôi đã trao đổi kỹ thuật được rồi; cái tôi đang cải thiện là nói trôi chảy mà không phải dừng lại dịch trong đầu.

---

## Thiết kế hệ thống

### Q066 — Batch chạy một lần mỗi ngày. Nếu nghiệp vụ yêu cầu mỗi 5 phút, bạn đổi gì?

> `batch` · độ khó 3/3

Câu query đổi trước tiên. Hiện nó lấy cả ngày; ở mức 5 phút tôi cần đọc tăng dần dùng watermark, ví dụ timestamp xử lý gần nhất. Thứ hai, cold start bắt đầu quan trọng, nên nếu mỗi lần chạy dưới 15 phút tôi sẽ cân nhắc Lambda. Thứ ba, các lần chạy có thể chồng nhau nên tôi giới hạn concurrency. Cuối cùng file output sẽ tự ghi đè, nên tôi thêm timestamp và làm job idempotent.

### Q067 — Thiết kế cách đơn giản đảm bảo file hằng đêm được gửi tới đối tác, kể cả khi có lỗi.

> `reliability` · độ khó 3/3

Job tạo file, upload, rồi ghi nhận thành công vào đâu đó — một dòng DynamoDB hoặc một object đánh dấu. Lỗi tạm thời thì retry kèm backoff. Nếu retry hết mà vẫn lỗi thì báo động thay vì im lặng thất bại. Phần bổ sung quan trọng là một kiểm tra riêng vào giờ hạn: nếu tới lúc đó chưa có bản ghi thành công thì cảnh báo. Cái đó bắt được trường hợp job không chạy lần nào, thứ mà alarm lỗi không phát hiện được.

### Q068 — Một team mới cần quyền đọc một prefix S3 từ account AWS khác. Bạn thiết lập thế nào?

> `security` · độ khó 3/3

Tôi sẽ tạo một role trong account của chúng tôi để account của họ assume — không chia sẻ access key. Trust policy ghi account của họ làm principal. Permission policy cấp s3:GetObject chỉ trên prefix đó, cộng ListBucket kèm điều kiện prefix, nếu không họ liệt kê được cả bucket. Nếu object được mã hóa thì họ cũng cần kms:Decrypt trên key. Sau đó tôi rà soát quyền định kỳ.

### Q069 — Hóa đơn AWS tháng này tăng 30%. Bạn điều tra thế nào?

> `cost` · độ khó 2/3

Tôi bắt đầu ở Cost Explorer, nhóm theo dịch vụ và so sánh tháng này với tháng trước. Điểm mấu chốt là tìm phần chênh lệch chứ không phải khoản lớn nhất — dịch vụ tốn nhiều nhất có thể vẫn bình thường. Rồi tôi nhóm theo usage type để xem cụ thể cái gì tăng. Nguyên nhân phổ biến là phí dữ liệu qua NAT, lượng log đổ vào CloudWatch, và tài nguyên bị bỏ quên. Sau đó tôi đặt budget alert.

### Q070 — Nếu làm lại migration MySQL sang PostgreSQL với gần như không downtime, bạn đổi gì?

> `migration` · độ khó 2/3

Thiết kế của chúng tôi là cutover có kế hoạch — restore snapshot, transform, load. Cách đó cần một khoảng đóng băng. Để gần như không downtime, tôi sẽ dùng DMS với change data capture: load toàn bộ trước, rồi để CDC giữ đích đồng bộ trong khi nguồn vẫn đang chạy. Lúc cutover ta dừng ghi một chút, chờ độ trễ về 0, rồi trỏ ứng dụng sang PostgreSQL. Đánh đổi là phức tạp hơn và kế hoạch rollback khó hơn.

---

## Networking Luồng đi & Concept

### Q071 — Giải thích luồng đi của một request từ khi user gõ domain đến khi chạm EC2.
> `networking` · độ khó 2/3
User gõ domain, trình duyệt hỏi DNS. Route 53 phân giải domain thành IP public và trả về cho client. Client gửi HTTP request tới IP đó, request đi qua internet và chạm Internet Gateway của VPC. Tại IGW, AWS làm 1:1 NAT dịch IP public thành IP private của EC2. Hạ tầng ngầm của AWS (VPC mapping service) nhìn IP private, biết nó thuộc subnet nào và đẩy gói tin vào subnet đó. Gói tin đi qua NACL ở tầng subnet, rồi qua Security Group ở tầng ENI của EC2, cuối cùng chạm hệ điều hành. Điểm quan trọng: chiều vào KHÔNG dùng route table của subnet — route table chỉ dùng cho chiều đi ra.

---

### Q072 — Route table có inbound và outbound không?
> `networking` · độ khó 1/3
Không. Route table không chia inbound/outbound như firewall. Nó chỉ có destination và target, và chỉ định tuyến cho chiều đi ra khỏi subnet. Route table trả lời câu hỏi "gói tin này đi đường nào", không phải "gói tin này được vào hay không". Việc lọc inbound/outbound là nhiệm vụ của NACL (tầng subnet) và Security Group (tầng instance). Chiều về của gói tin không cần cấu hình route table vì hạ tầng AWS tự biết IP đích thuộc subnet nào nhờ local route.

---

### Q073 — Phân biệt Destination và Target trong route table.
> `networking` · độ khó 1/3
Destination là dải IP đích mà gói tin muốn tới. Target là "cổng thoát" xử lý và vận chuyển gói tin đó. Ví dụ: destination 0.0.0.0/0 với target là Internet Gateway nghĩa là mọi IP ngoài VPC sẽ đi ra qua IGW. Destination 10.0.0.0/16 với target local nghĩa là traffic nội bộ VPC đi thẳng trong mạng. AWS dùng nguyên tắc longest prefix match: nếu một IP khớp nhiều dòng, dòng nào có CIDR chi tiết hơn sẽ thắng.

---

### Q074 — Trong cùng một VPC, có cần route table để hai subnet nói chuyện với nhau không?
> `networking` · độ khó 2/3
Có, vẫn cần. Nhưng AWS tự tạo sẵn dòng local route mặc định (VD 10.0.0.0/16 → local) trong mọi route table, nên ta không phải cấu hình tay. Nếu không có dòng local này, các subnet trong VPC sẽ không tìm thấy nhau. Nhiều người tưởng không cần route table vì AWS làm sẵn. Security Group không thay thế được route table: route table lo tìm đường đi, Security Group lo kiểm tra quyền ra vào. Hai việc hoàn toàn khác nhau.

---

### Q075 — Khi request từ internet đi qua IGW vào VPC, IGW dựa vào đâu để biết đưa gói tin tới subnet nào?
> `networking` · độ khó 3/3
IGW không dùng route table của subnet cho chiều vào. Nó dựa vào VPC mapping service — bản đồ ánh xạ IP toàn VPC. IP public mà Route 53 trả về không nằm trực tiếp trên EC2; EC2 chỉ có IP private. AWS dùng 1:1 NAT để liên kết public IP với private IP, và bản đồ này lưu ở quy mô toàn VPC. Khi gói tin chạm IGW, IGW dịch public IP thành private IP, rồi hạ tầng AWS đối chiếu private IP với dải các subnet để biết đẩy vào đâu. Route table của subnet hoàn toàn không tham gia chiều vào.

---

### Q076 — Vậy route table trỏ 0.0.0.0/0 → IGW dùng để làm gì, nếu chiều vào không dùng nó?
> `networking` · độ khó 2/3
Dòng đó chỉ dùng cho chiều đi ra. Khi EC2 trong subnet muốn phản hồi request cho user internet, hoặc chủ động gọi ra ngoài, nó ném gói tin ra IGW theo dòng 0.0.0.0/0. IGW dịch ngược private IP thành public IP để gửi trả. Route table trong AWS bản chất là công cụ định tuyến một chiều — chỉ trả lời "tôi muốn gửi tới IP X thì đi đường nào". Chiều vào là việc của VPC mapping service, NACL và Security Group.

---

### Q077 — Sự khác nhau giữa Security Group và NACL?
> `networking` · độ khó 2/3
Security Group gắn ở tầng ENI của instance, NACL gắn ở tầng subnet. Khác biệt quan trọng nhất: Security Group là stateful — nếu cho traffic vào thì chiều trả về tự động được phép ra. NACL là stateless — mỗi gói tin xét độc lập, phải viết rule cho cả hai chiều. Security Group chỉ allow được, NACL vừa allow vừa deny tường minh và xét theo số thứ tự rule, rule khớp đầu tiên thắng. Thực tế dùng Security Group cho công việc hằng ngày, NACL khi cần chặn nguyên một dải IP ở cấp subnet.

---

### Q078 — NACL đã allow inbound 443 và outbound 443, nhưng client gọi HTTPS vào EC2 vẫn bị treo. Vì sao?
> `networking` · độ khó 3/3
Vấn đề là ephemeral port. Khi client mở kết nối, nó dùng một source port ngẫu nhiên thường trên 1024, ví dụ 54321. Chiều vào port đích là 443, NACL cho qua. Nhưng khi EC2 phản hồi, gói trả về đi tới port 54321 của client chứ không phải 443. NACL outbound chỉ allow 443 nên chặn gói này lại. Kết quả là request vào được nhưng response không ra được, gây treo. Security Group không gặp lỗi này vì nó stateful. Cách sửa: thêm outbound rule allow TCP 1024-65535. AWS khuyến nghị dải này để phủ hết ephemeral port của Linux, Windows và ELB.

---

### Q079 — Tại sao ALB phải dùng Alias record thay vì A record hoặc CNAME?
> `networking` · độ khó 2/3
ALB không có IP cố định — AWS thay đổi IP liên tục để auto-scale và đảm bảo tính sẵn sàng. Nên không thể hardcode IP vào A record. CNAME thì không dùng được cho apex domain (domain gốc như example.com), chỉ dùng cho subdomain. Alias là bản ghi riêng của Route 53, trỏ trực tiếp tới AWS resource, dùng được cả cho apex domain lẫn subdomain, và không tính phí query DNS. Route 53 tự phân giải Alias thành IP hiện tại của ALB tại thời điểm truy vấn.

---

### Q080 — ALB có 2 IP ở 2 AZ. Bạn tắt hết EC2 ở AZ-A nhưng AZ-A vẫn hoạt động. dig domain nhận mấy IP?
> `networking` · độ khó 3/3
Vẫn nhận 2 IP. Đây là điểm nhiều người nhầm. Có hai tầng độc lập: tầng DNS phản ánh node ALB nào còn sống, còn tầng ALB phản ánh target nào còn khỏe. Node ALB ở AZ-A vẫn sống nên DNS vẫn trả IP của nó. Khi node đó nhận request, nó sẽ forward sang target khỏe ở AZ-B nhờ cross-zone load balancing. DNS chỉ bớt IP khi chính node ALB ở AZ-A chết, ví dụ khi cả AZ sập. Nếu tất cả target ở mọi AZ đều unhealthy, ALB chuyển sang fail-open, vẫn nhận và forward thay vì từ chối.

---

### Q081 — Cross-Zone Load Balancing hoạt động thế nào?
> `networking` · độ khó 3/3
Khi bật, một node ALB ở bất kỳ AZ nào sẽ phân phối request đều tới tất cả target khỏe mạnh ở mọi AZ, không phân biệt AZ. Nghĩa là node ALB ở AZ-A có thể gửi request tới target ở AZ-B ngay cả khi target AZ-A vẫn khỏe — đây là hành vi mặc định, không phải cơ chế cứu hộ khi quá tải. Với ALB, cross-zone luôn bật ở cấp load balancer và không tắt được, chỉ tinh chỉnh được ở cấp target group. Với NLB thì mặc định tắt. Lưu ý: traffic cross-AZ bị tính phí data transfer.

---

### Q082 — DNS caching ảnh hưởng thế nào đến failover?
> `networking` · độ khó 3/3
Client không hỏi DNS mỗi request. Kết quả DNS được cache ở nhiều tầng: trình duyệt, hệ điều hành, và resolver của ISP, theo giá trị TTL. Hệ quả là một client có thể dùng cùng một IP suốt một thời gian dài. Khi một AZ sập, client đã cache IP cũ vẫn cố kết nối vào IP chết cho tới khi TTL hết hạn. Vì vậy DNS không bao giờ là cơ chế failover tốt — nó chậm và phụ thuộc cache ngoài tầm kiểm soát. Failover thật nên dựa vào ALB, health check, hoặc CloudFront origin retry, không dựa vào việc đổi DNS record.

---

### Q083 — ALB forward request tới EC2. Trong Security Group của EC2, bạn allow inbound từ đâu?
> `networking` · độ khó 2/3
Allow inbound từ Security Group của ALB, không phải từ IP. Lý do là IP của ALB thay đổi liên tục khi auto-scale; nếu hardcode IP thì hôm nay đúng, mai ALB scale ra IP mới sẽ bị chặn. Tham chiếu bằng Security Group ID thì AWS tự resolve — bất kỳ ENI nào gắn SG đó của ALB đều được phép. Đây là best practice: dùng SG reference thay vì IP range cho traffic nội bộ giữa các tier.

---

### Q084 — Có CloudFront đứng trước ALB. Làm sao bắt buộc mọi traffic phải đi qua CloudFront, không cho gọi thẳng ALB?
> `networking` · độ khó 3/3
OAC chỉ dùng được cho S3, không dùng cho ALB. Với ALB dùng hai cách kết hợp. Thứ nhất, CloudFront thêm một custom header bí mật (ví dụ X-Origin-Verify) vào mọi request tới origin; ALB listener rule kiểm tra header đó, khớp thì forward, không khớp thì trả 403. Thứ hai, giới hạn Security Group của ALB chỉ nhận traffic từ prefix list của CloudFront (com.amazonaws.global.cloudfront.origin-facing). Prefix list một mình chưa đủ vì mọi CloudFront distribution trên thế giới đều nằm trong đó, nên custom header là phần xác thực đúng distribution của mình. Chỉ giấu DNS name của ALB không phải là bảo vệ — đó là security through obscurity.

---

### Q085 — EC2 ở private subnet gọi tới S3. Route table có 0.0.0.0/0 → NAT Gateway. Gói tin đi đường nào? Thêm S3 Gateway Endpoint thì đổi gì?
> `networking` · độ khó 2/3
M��c định gói tin đi qua NAT Gateway rồi ra internet để tới S3, tốn phí NAT data processing. Khi thêm S3 Gateway Endpoint, AWS tự thêm một route vào route table với destination là prefix list của S3 (pl-xxxx) và target là endpoint (vpce-xxxx). Vì prefix list cụ thể hơn 0.0.0.0/0, theo longest prefix match, traffic tới S3 sẽ đi qua endpoint thay vì NAT. Lợi ích: không tốn phí NAT, và traffic không rời mạng AWS nên an toàn hơn. Gateway Endpoint miễn phí và chỉ dùng cho S3 và DynamoDB.

---

### Q086 — Phân biệt Gateway Endpoint và Interface Endpoint.
> `networking` · độ khó 3/3
Gateway Endpoint chỉ dùng cho S3 và DynamoDB, miễn phí, hoạt động bằng cách thêm route vào route table trỏ tới prefix list của dịch vụ. Interface Endpoint dùng cho hầu hết các dịch vụ AWS khác, hoạt động bằng cách tạo một ENI với IP private trong subnet, và tính phí theo giờ cộng phí data. Điểm chung là cả hai cho phép resource trong private subnet truy cập dịch vụ AWS mà không cần đi qua internet, giúp tránh phí NAT và giữ traffic trong mạng AWS. Chọn Gateway khi chỉ cần S3/DynamoDB vì nó miễn phí; các dịch vụ còn lại buộc dùng Interface.

---

### Q087 — Không có CloudFront, Route 53 trỏ thẳng tới S3 được không?
> `networking` · độ khó 2/3
Được, nhưng có điều kiện. Chỉ trỏ thẳng tới S3 khi bật Static Website Hosting trên bucket, và tên bucket phải trùng khớp hoàn toàn với domain — muốn dùng example.com thì bucket phải tên chính xác là example.com. Dùng Alias A record trỏ tới S3 website endpoint. Hạn chế lớn nhất: S3 website endpoint chỉ chạy HTTP, không hỗ trợ HTTPS. Muốn có HTTPS bắt buộc phải đặt CloudFront phía trước. Vì vậy trong thực tế, hầu hết dùng Route 53 → CloudFront → S3 (qua OAC) để vừa có HTTPS vừa bảo vệ được bucket.

---

### Q088 — Giải thích Control Plane và Data Plane qua ví dụ ALB.
> `networking` · độ khó 3/3
Control plane là các hoạt động chạy nền, liên tục, không gắn với từng request: ELB service theo dõi node nào còn sống, ALB health-check các target mỗi chu kỳ, và cập nhật DNS record của ALB. Data plane là đường đi của từng request thật: client hỏi DNS nhận IP, gửi TCP tới IP đó, node ALB nhận rồi chọn target forward. Hai tầng này độc lập hoàn toàn — Route 53 không hỏi ALB gì theo từng request, ALB cũng không báo cáo cho Route 53. Hiểu tách bạch hai tầng này giúp trả lời đúng nhiều tình huống, ví dụ vì sao tắt EC2 mà DNS vẫn trả đủ IP.

---

## Backup & Recovery (bổ sung)

### Q089 — Đã có PITR 35 ngày, tại sao còn cần daily backup 30 ngày trong vault? Nghe như trùng lặp.
> `backup` · độ khó 3/3
Về thời gian thì 35 ngày bao trùm 30 ngày, nhưng hai cái lưu ở hai nơi khác nhau nên không trùng. PITR nằm bên trong chính RDS instance — nó là snapshot cộng transaction log gắn liền với instance. Vault backup nằm ở một kho tách biệt, ngoài RDS. Sự khác biệt lộ ra khi cả instance bị xóa hoặc hỏng: PITR biến mất theo instance vì nó nằm trong đó, còn vault backup vẫn còn nguyên. Nên chúng bảo vệ hai loại sự cố khác nhau — PITR cho lỗi logic khi instance vẫn sống, vault cho thảm họa khi mất luôn instance. Sự chồng lấn về thời gian là có chủ đích: trong 30 ngày gần nhất ta muốn có cả hai lớp bảo vệ. Câu chốt: they are layers, not duplicates.

---

### Q090 — Hacker xóa sạch RDS instance lúc 3h chiều. PITR bật, retention 35 ngày. Daily vault backup gần nhất là 2h sáng. Khôi phục được tới đâu?
> `backup` · độ khó 3/3
Chỉ khôi phục được về bản vault 2h sáng, mất khoảng 13 tiếng dữ liệu. Không dùng được PITR dù nó còn 35 ngày, vì PITR nằm bên trong chính instance đã bị xóa — instance chết thì PITR chết theo. Vault backup nằm ở kho riêng nên sống sót. Nếu vault có bật Vault Lock thì kể cả hacker có quyền admin cũng không xóa được bản backup đó. Tình huống này cho thấy đúng lý do phải có backup tách biệt: PITR chính xác tới giây nhưng không chịu được việc mất cả instance, còn vault kém chính xác hơn nhưng bền hơn trước thảm họa.

---

### Q091 — Phân biệt PITR và daily/monthly backup về đơn vị khôi phục.
> `backup` · độ khó 2/3
PITR cho phép khôi phục tới bất kỳ thời điểm nào trong khoảng retention, chính xác tới giây, nhờ giữ transaction log liên tục cộng với snapshot. Daily và monthly backup chỉ khôi phục về các mốc rời rạc — mỗi ngày một bản, mỗi tháng một bản. Ví dụ nếu xóa nhầm dữ liệu lúc 14:37, PITR cho khôi phục về 14:36:59 chỉ mất vài giây dữ liệu, còn daily backup chỉ về được 2h sáng nên mất cả buổi. Đổi lại, PITR chỉ giữ ngắn hạn (RDS tối đa 35 ngày), còn monthly backup giữ được nhiều năm cho compliance. Vì vậy dùng phân tầng: PITR cho độ chính xác ngắn hạn, backup cho độ bền dài hạn.

---

### Q092 — Backup RDS giữ 5 năm, sao không export ra S3 Glacier cho rẻ mà lại để trong Backup Vault?
> `backup` · độ khó 3/3
Vì backup 5 năm là để cứu hộ và compliance, nên khi cần tới thường là lúc khẩn cấp hoặc bị audit, và lúc đó mình muốn một cú restore ra được database hoàn chỉnh chứ không phải ngồi dựng lại pipeline import từ Glacier. Snapshot trong vault giữ nguyên định dạng gốc — index, constraint, tính nhất quán — còn nếu dump ra CSV nhét vào Glacier thì khi restore phải build lại schema và index, với dữ liệu 5 năm tuổi thì rất rủi ro. Ngoài ra vault có Vault Lock để chống xóa, phục vụ compliance. Về chi phí, warm storage của AWS Backup tính theo kiểu incremental — chỉ tính phần dung lượng thay đổi giữa các bản chứ không tính trọn từng snapshot — nên không đắt như tưởng. Với dự án của mình, mình đã tính ra khoảng 13 đô một tháng và khách hàng đồng ý. Nên đây là quyết định cân nhắc trade-off giữa chi phí và khả năng khôi phục, không phải bị giới hạn ép buộc.

---

### Q093 — Tại sao log lại lưu trên S3 mà không lưu trong Backup Vault như database?
> `backup` · độ khó 2/3
Vì database và log phục vụ hai mục đích khác nhau. Database backup cần restore ra được một instance hoàn chỉnh khi có sự cố, nên để trong Backup Vault để restore trực tiếp và nhất quán. Log thì chỉ cần đọc lại để tra cứu hoặc phân tích, không cần restore ra database sống. Với nhu cầu chỉ đọc, lưu trên S3 hợp hơn nhiều: rẻ hơn, tier được qua Standard-IA và Glacier theo lifecycle, và quan trọng là query trực tiếp được bằng Athena mà không cần restore gì cả. Backup Vault không thiết kế cho việc query dữ liệu kiểu này. Nên nguyên tắc thiết kế xuyên suốt là: cần restore ra database thì dùng vault, chỉ cần đọc lại thì dùng S3.

---

