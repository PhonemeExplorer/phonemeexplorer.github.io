# ARTIC-48: Một Chỉ Số Hoạt Hình Khuôn Mặt Theo Cấu Âm

## 1. Tên & Tiền Đề

**ARTIC-48** — *Articulatory Representation for Translingual Inference & Coarticulation, 48 kênh.*

Luận điểm cốt lõi: hoạt hình khuôn mặt khi nói không nên được tham số hóa theo *cơ bắp làm gì* (MetaHuman 135) hay *ngôn ngữ trông như thế nào* (viseme), mà phải theo **những gì bộ máy phát âm đang thực hiện**. Mọi âm thanh được nói ra trên Trái Đất đều được tạo ra bởi một tập hữu hạn các bộ cấu âm (môi, hàm, thân lưỡi, đầu lưỡi, màn hầu, thanh quản) hoạt động theo một số gradient liên tục nhỏ. ARTIC-48 biến những gradient đó thành biểu diễn chính, sau đó render xuống các đầu ra theo rig cụ thể.

Điều này phù hợp với cấu trúc của IPA — vốn *có tính đặc trưng một phần* — 107 ký tự phân đoạn của nó được tổ chức theo các trục cấu âm trực giao (vị trí, phương thức, hữu thanh, tròn môi, chiều cao, độ lùi). ARTIC-48 biến các trục đó thành các kênh theo nghĩa đen.

---

## 2. Sơ Đồ Khái Niệm

```
                           ┌──────────────────────────┐
        VĂN BẢN ──► G2P ──►│                          │
                           │   Luồng IPA + Ngữ Điệu   │
        ÂM THANH ──► ASR ──►│   (âm vị + F0 + năng lượng)│
                           └────────────┬─────────────┘
                                        │
                                        ▼
                  ┌────────────────────────────────────────┐
                  │  Ánh xạ A: Phân Giải Cấu Âm           │
                  │     (tra bảng đặc trưng tất định)      │
                  └────────────────────┬───────────────────┘
                                       ▼
        ┌──────────────────────────────────────────────────────────┐
        │                       ARTIC-48                           │
        │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │
        │  │ Tầng S   │ │ Tầng P   │ │ Tầng E   │ │ Tầng R   │    │
        │  │ Lời nói  │ │Ngữ điệu  │ │ Cảm xúc  │ │ Phần dư  │    │
        │  │  24 kênh │ │  6 kênh  │ │ 12 kênh  │ │  6 kênh  │    │
        │  └──────────┘ └──────────┘ └──────────┘ └──────────┘    │
        │    Cấu âm + Hữu thanh + Cảm xúc + Chuyển động phụ      │
        └──────────────────────┬───────────────────────────────────┘
                               │
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                  ▼
   Ánh xạ B: ma trận    Ánh xạ C: MLP       Dùng trực tiếp
   blendshape tuyến tính theo rig cụ thể    bởi mô hình
   → MetaHuman 135      → rig phong cách    chuyển động
            ▼                  ▼
   ARKit 52 + 83 mở rộng  Pixar / VTuber /
                           thú vật / không lưỡi
```

Bốn **tầng** là ý tưởng trụ cột: lời nói, ngữ điệu, cảm xúc và phần dư là các *không gian con có thể tách biệt*. Một mô hình audio-to-animation có thể dự đoán Tầng S+P từ âm học đơn thuần. Một mô hình text-to-motion có thể điều khiển Tầng E độc lập. Một pipeline thu capture biểu diễn có thể lấp đầy cả bốn. Sự tách biệt là cấu trúc, không phải học được.

---

## 3. Định Nghĩa Các Kênh

48 kênh được tổ chức thành bốn tầng. Mỗi kênh có ngữ nghĩa cấu âm hoặc cảm xúc được định nghĩa rõ ràng, dải giá trị, và hồ sơ tương tác đã biết.

### Tầng S — Cấu Âm Lời Nói (24 kênh)

Đây là các kênh được điều khiển bởi nội dung âm vị. Chúng phản chiếu trực tiếp các chiều tổ chức của IPA.

| # | Kênh | Bộ cấu âm | Dải | Ngữ nghĩa |
|---|---|---|---|---|
| S01 | `jaw_open` | hàm dưới | 0–1 | Xoay hàm theo chiều dọc; proxy chiều cao nguyên âm |
| S02 | `jaw_protrude` | hàm dưới | -1 đến 1 | Dịch chuyển hàm ra trước/ra sau |
| S03 | `jaw_lateral` | hàm dưới | -1 đến 1 | Dịch chuyển hàm trái/phải (chủ yếu cảm xúc/nghỉ ngơi) |
| S04 | `lip_aperture` | môi | 0–1 | Khoảng cách môi theo chiều dọc, độc lập với hàm |
| S05 | `lip_rounding` | vòng miệng | 0–1 | Phồng + co tròn (/u/, /o/) |
| S06 | `lip_spreading` | cơ mím | 0–1 | Kéo ngang (/i/, /e/, gần với nụ cười) |
| S07 | `lip_compression` | vòng miệng | 0–1 | Áp lực khép hai môi (/p/, /b/, /m/) |
| S08 | `lower_lip_tuck` | cơ cằm + OO | 0–1 | Tiếp xúc môi-răng (/f/, /v/) |
| S09 | `upper_lip_raise` | LLS | 0–1 | Lộ răng trên |
| S10 | `lip_asymmetry` | OO hai bên | -1 đến 1 | Chênh lệch T/P, nghỉ ngơi + nhấn mạnh |
| S11 | `tongue_height` | thân lưỡi | 0–1 | Vị trí cao↔thấp (nguyên âm khép↔mở) |
| S12 | `tongue_backness` | thân lưỡi | 0–1 | Vị trí trước↔sau (trục /i/↔/u/) |
| S13 | `tongue_tip_raise` | đầu lưỡi | 0–1 | Tiếp xúc răng huyệt/nha (/t/, /d/, /n/, /l/) |
| S14 | `tongue_tip_curl` | đầu lưỡi | 0–1 | Cuộn lưỡi ngược (/ɖ/, /ʂ/, /ɹ/ tiếng Anh) |
| S15 | `tongue_groove` | lưng lưỡi | 0–1 | Rãnh cho âm xát (/s/, /ʃ/, /z/) |
| S16 | `tongue_dorsum_raise` | gốc lưỡi | 0–1 | Tiếp xúc màn hầu (/k/, /g/, /ŋ/) |
| S17 | `tongue_root_retract` | gốc lưỡi | 0–1 | Hầu họng/ATR (tiếng Ả Rập /ʕ/, ngôn ngữ ±ATR) |
| S18 | `tongue_protrusion` | toàn lưỡi | 0–1 | Nha/liên nha (/θ/, /ð/) |
| S19 | `velum_lower` | vòm mềm | 0–1 | Kết nối mũi (/m/, /n/, /ŋ/, nguyên âm mũi) |
| S20 | `larynx_height` | thanh quản | -1 đến 1 | Nâng (căng) ↔ hạ (nguyên âm tròn) |
| S21 | `voicing` | thanh môn | 0–1 | Hữu thanh ↔ vô thanh; tạo rung dưới da |
| S22 | `aspiration` | thanh môn | 0–1 | Âm bật hơi/thở (/pʰ/) |
| S23 | `cheek_compression` | cơ má | 0–1 | Căng má tống khí/bật hơi |
| S24 | `nostril_flare` | cơ giãn mũi | 0–1 | Chỉ báo luồng khí mũi |

Các nhóm bộ cấu âm — *hàm, môi, lưỡi, màn hầu, thanh quản* — có cấu trúc **phân cấp**: mô hình có thể được huấn luyện hoặc ràng buộc ở cấp nhóm (ví dụ "tắt hết kênh lưỡi cho nhân vật không có lưỡi") mà không cần chạm vào từng kênh riêng lẻ. Trong mỗi nhóm, các kênh *phần lớn* tuyến tính và có thể kết hợp cộng tính. Phi tuyến tính chính là giữa `jaw_open` và `lip_aperture` — chúng ràng buộc lẫn nhau (môi không thể tách rộng hơn độ mở hàm cộng một khoảng trượt nhỏ).

### Tầng P — Ngữ Điệu (6 kênh)

Thông tin siêu phân đoạn điều biến toàn bộ khuôn mặt.

| # | Kênh | Dải | Ngữ nghĩa |
|---|---|---|---|
| P01 | `f0_normalized` | -1 đến 1 | Cao độ tương đối so với mức cơ sở của người nói; kết hợp lông mày + thanh quản |
| P02 | `energy` | 0–1 | Độ lớn từ RMS; khuếch đại chuyển động hàm |
| P03 | `tempo_local` | 0–1 | Tốc độ nói cục bộ; điều chỉnh cửa sổ đồng cấu âm |
| P04 | `stress_marker` | 0–1 | Trọng âm từ/ngữ; đỉnh ở âm tiết được nhấn |
| P05 | `boundary_marker` | 0–1 | Ranh giới ngữ đoạn; giải phóng căng thẳng, cho phép chớp mắt/thở |
| P06 | `breath_phase` | -1 đến 1 | Hít vào ↔ thở ra; kết hợp ngực/mũi/vai |

### Tầng E — Cảm Xúc & Biểu Cảm (12 kênh)

Độc lập với nội dung lời nói. Có thể được thiết lập bởi bộ điều khiển cảm xúc, chỉnh sửa bởi animator, hoặc suy ra từ cảm xúc văn bản/âm thanh.

| # | Kênh | Dải | Ngữ nghĩa |
|---|---|---|---|
| E01 | `valence` | -1 đến 1 | Cảm xúc tích cực↔tiêu cực |
| E02 | `arousal` | 0–1 | Mức độ kích hoạt |
| E03 | `dominance` | -1 đến 1 | Tư thế phục tùng↔thống trị |
| E04 | `brow_raise` | 0–1 | Kích hoạt cơ trán (ngạc nhiên, hỏi) |
| E05 | `brow_furrow` | 0–1 | Cơ nhăn mày (lo lắng, tập trung) |
| E06 | `eye_widen` | 0–1 | Nâng mí mắt (tỉnh táo) |
| E07 | `eye_squint` | 0–1 | Cơ vòng mắt (Duchenne, dò xét) |
| E08 | `smile` | 0–1 | Kéo gò má; trực giao với S06 kéo môi ngang |
| E09 | `frown` | 0–1 | DAO + cơ cằm (buồn, ghê tởm) |
| E10 | `sneer` | 0–1 | LLSAN (ghê tởm, khinh thường) |
| E11 | `lip_press` | 0–1 | Ép môi căng thẳng (kìm nén, tức giận) |
| E12 | `gaze_engagement` | 0–1 | Nhìn thẳng ↔ né tránh ánh mắt |

Lưu ý rằng E08 (`smile`) và S06 (`lip_spreading`) được cố tình tách biệt mặc dù đầu ra MetaHuman của chúng chồng lên nhau — kéo môi ngang theo ngữ âm (nói "cheese") khác biệt về mặt cấu âm so với mỉm cười theo cảm xúc (liên quan đến Duchenne ở E07). Tách biệt chúng là một bước cải tiến lớn so với các hệ thống hiện tại.

### Tầng R — Chuyển Động Phụ & Phần Dư (6 kênh)

Những thứ là hệ quả vật lý hơn là tín hiệu điều khiển, được giữ tách biệt để có thể tạo ra theo quy trình xuôi dòng.

| # | Kênh | Dải | Ngữ nghĩa |
|---|---|---|---|
| R01 | `skin_compression_jaw` | 0–1 | Nếp gấp dưới cằm |
| R02 | `skin_compression_cheek` | 0–1 | Sâu thêm rãnh mũi môi |
| R03 | `neck_tension` | 0–1 | Kích hoạt cơ ức đòn chũm/bám da cổ |
| R04 | `swallow_phase` | 0–1 | Thanh quản lên + lưỡi lùi thoáng qua |
| R05 | `micro_tremor` | 0–1 | Bao rung sinh lý tần số cao |
| R06 | `idle_drift` | 0–1 | Bao dao động tư thế tần số thấp |

Các kênh Tầng R thường *không được dự đoán* bởi mô hình âm thanh — chúng được tổng hợp từ các tầng S/P/E qua các quy tắc vật lý đơn giản hoặc đường cong phong cách. Đây là lý do ARTIC-48 chỉ có 48 chiều thay vì 80 chiều.

---

## 4. Các Pipeline Ánh Xạ

### Ánh xạ A: IPA → ARTIC-48 (Tất Định + Làm Mịn Học Máy)

Mỗi phân đoạn IPA có một **vector đích chuẩn** trong Tầng S, được suy ra từ bó đặc trưng cấu âm của nó. Đây là bảng tra cứu cố định được xây dựng từ lý thuyết ngữ âm học, không phải học được. Nó chứa khoảng 110 mục (107 ký tự IPA + các biến thể theo dấu phụ cho mũi hóa, độ dài, ATR).

Việc tra cứu là tất định. Điều được *học* là **bộ tạo quỹ đạo** biến chuỗi vector đích thành tín hiệu 48 kênh liên tục, tính đến đồng cấu âm, đặt âm thiếu, và định thời theo ngôn ngữ cụ thể. Phần đó là một Transformer nhỏ hoặc hệ thống lò xo giảm xóc tắt dần tới hạn kiểu Task Dynamics (xem §5).

### Ánh xạ B: ARTIC-48 → MetaHuman 135 (Tuyến Tính, Đã Hiệu Chỉnh)

Ánh xạ B là một **ma trận trọng số 48 × 135** cộng với một tập nhỏ phi tuyến tính trên từng đích (tích cặp cho ghép hàm/môi, bão hòa sigmoid ở các cực trị). Nó được hiệu chỉnh một lần cho mỗi danh tính bằng hồi quy ghép cặp trên tập dữ liệu tham chiếu nhỏ (~30 phút thu capture biểu diễn MetaHuman có đồng bộ âm thanh).

Quan trọng là, ma trận này **phần lớn thưa thớt**: `S01 jaw_open` kích hoạt `jawOpen` và một số curve phụ thuộc; `S05 lip_rounding` kích hoạt `mouthFunnel`, `mouthPucker`, và yếu hơn là `cheekPuff`. Mỗi kênh ARTIC-48 chạm vào khoảng 3–8 trong số 135 curve. Đây là điều làm cho ánh xạ có thể diễn giải được — bạn có thể đọc ma trận trọng số và xác minh nó.

### Ánh xạ C: ARTIC-48 → Rig Tùy Chỉnh (MLP Học Được, Nhận Thức Cấu Trúc)

Đối với các rig không phải MetaHuman (phong cách Pixar, thú vật, sinh vật phi giải phẫu, VTuber 2D), Ánh xạ C là một MLP nhỏ theo từng rig (thường ~500K tham số, 3 lớp ẩn) được huấn luyện trên:

- Vài phút dữ liệu ghép cặp ARTIC-48 → rig được animator thu, hoặc
- Các cặp tổng hợp tạo ra bằng cách retarget biểu diễn MetaHuman qua Ánh xạ B đảo ngược, sau đó chỉnh tay.

Cấu trúc phân cấp của Tầng S là then chốt ở đây: nhân vật không có lưỡi (minion Pixar, cá) chỉ cần định tuyến các kênh lưỡi (S11–S18) về zero trong MLP, và các kênh môi/hàm chịu toàn bộ gánh nặng lời nói. Mô hình không cần học điều đó — đó là quyết định đấu dây.

### Chiến Lược Nén

Việc giảm từ 135 → 48 là *phân giải có cấu trúc*, không phải PCA. Lựa chọn này có chủ đích:

- **PCA** sẽ cho không gian 48 chiều tối ưu cho tái tạo nhưng không thể diễn giải — các kênh sẽ trộn lẫn lời nói và cảm xúc, đúng điều chúng ta muốn tránh.
- **VAE học được** thậm chí còn tệ hơn cho khả năng diễn giải và sẽ ghép nối lời nói với bất kỳ phân phối cảm xúc nào trong tập huấn luyện.
- **Phân giải cấu âm** đánh đổi một lượng nhỏ độ trung thực tái tạo (≈3–5% lỗi mỗi frame so với PCA-48) lấy khả năng diễn giải đầy đủ, khả năng chỉnh sửa thủ công, và tách biệt đúng.

Đây là đánh đổi đúng đắn. Animator cần có thể nắm `lip_rounding` và giảm nó xuống. Kênh PCA #17 không thể chỉnh sửa.

---

## 5. Hành Vi Thời Gian

Đồng cấu âm — thực tế rằng /k/ trong "key" trông khác /k/ trong "coo" — là phần khó nhất của hoạt hình lời nói. ARTIC-48 xử lý nó bằng **kết hợp hai giai đoạn**:

**Giai đoạn 1: Lớp động học cấu âm.** Mỗi kênh Tầng S được điều hành bởi hệ thống bậc hai tắt dần tới hạn với các hằng số thời gian theo từng kênh, theo tinh thần của Task Dynamics (Saltzman & Munhall 1989). Các bộ cấu âm nhanh (đầu lưỡi: ~30ms) đạt đích nhanh; các bộ cấu âm chậm (hàm: ~80ms; tròn môi: ~100ms) trễ lại. Điều này tạo ra sự đặt âm thiếu và chuyển động dự đoán đúng *miễn phí*, không cần tham số học được. Nó cũng thực thi ràng buộc vật lý rằng bạn không thể cử động hàm nhanh hơn giới hạn vật lý cho phép, ngăn chặn đầu ra giật cục phổ biến trong các hệ thống điều khiển bằng âm thanh.

**Giai đoạn 2: Tinh chỉnh nơ-ron.** Một Transformer nhỏ (~6 lớp, nhân quả hoặc với lookahead giới hạn cho thời gian thực) nhận tín hiệu đã làm mịn cấu âm cộng đặc trưng ngữ điệu và âm thanh, và dự đoán phần dư. Điều này nắm bắt các mẫu định thời theo ngôn ngữ cụ thể và theo người nói mà lớp động học không thể — ví dụ định thời mora của tiếng Nhật so với định thời trọng âm tiếng Anh, hoặc thói quen ép môi trước phụ âm tắc của một diễn viên cụ thể.

Đối với các pipeline không thời gian thực (render offline, phim), Giai đoạn 2 có thể là hai chiều và đầu ra tốt hơn đáng kể. Đối với thời gian thực (game, VTuber, agent hội thoại), lớp động học đơn thuần tạo ra đầu ra chấp nhận được với lookahead ~20ms.

Tầng R (phần dư/phụ) được tạo ra bởi các quy tắc thủ tục trên S+P+E: nén da theo chuyển động hàm + má với độ trễ nhỏ, căng cổ theo năng lượng, rung vi mô là nhiễu biên độ cố định được điều biến bởi kích thích.

---

## 6. Tích Hợp Đa Phương Thức

ARTIC-48 được thiết kế là một **biểu diễn trung tâm** mà tất cả các phương thức đầu vào đều chiếu vào.

- **Đầu vào âm thanh:** Bộ mã hóa kiểu wav2vec2 dự đoán Tầng S + Tầng P trực tiếp từ dạng sóng. Tầng E có thể được dự đoán từ các đặc trưng cảm xúc âm thanh (cao độ, năng lượng, chất lượng giọng) nhưng thường để cho bộ điều khiển cảm xúc riêng.
- **Đầu vào văn bản:** Grapheme-to-phoneme (G2P) → luồng IPA → Ánh xạ A → Tầng S + P. Các mô hình ngữ điệu TTS tiêu chuẩn dự đoán P. Cảm xúc từ văn bản lấp đầy E.
- **Điều khiển cảm xúc:** Các kênh Tầng E có thể được thiết lập bên ngoài bởi hệ thống hội thoại, đạo diễn cảnh, hoặc bộ điều khiển VAD (valence-arousal-dominance). Chúng là *cộng tính* với lời nói — nói "hello" trong khi buồn chỉ có nghĩa là E01 âm trong khi Tầng S theo quỹ đạo /h ɛ l oʊ/ bình thường. Tính kết hợp này là phần thưởng cho việc giữ các tầng tách biệt.
- **Chuyển giao đa ngôn ngữ:** Vì Tầng S là cấu âm, huấn luyện trên tiếng Anh dạy mô hình `tongue_tip_raise` trông như thế nào trong tiếp xúc răng huyệt — và kiến thức đó chuyển giao trực tiếp sang tiếng Tây Ban Nha, Hindi, Quan Thoại, hay bất kỳ ngôn ngữ nào có phụ âm tắc răng huyệt. Các hiện tượng theo ngôn ngữ cụ thể (thanh điệu tiếng Quan Thoại, âm tắc lưỡi trong tiếng Xhosa, nguyên âm mũi tiếng Pháp) được xử lý bởi ký hiệu IPA hiện có đưa vào cùng các kênh (thanh điệu → P01, âm tắc lưỡi → bước nhảy S07 + S15, nguyên âm mũi → S19 kéo dài). Không cần mô hình theo từng ngôn ngữ.

---

## 7. Khả Năng Mở Rộng

**Huấn luyện trên 100k+ video:** ARTIC-48 là biểu diễn đích tự nhiên cho các mô hình audio-to-animation tự giám sát. Không gian 48 chiều đủ nhỏ để ngay cả các mô hình cỡ vừa cũng có thể khớp tốt, và cấu trúc tách biệt có nghĩa là bạn có thể huấn luyện trên dữ liệu chưa tinh chỉnh — người nói đang mỉm cười không làm ô nhiễm các kênh lời nói vì cảm xúc nằm trong Tầng E, nơi nó tự do thay đổi độc lập.

**Bền vững qua các cấu trúc khuôn mặt:** Vì chỉ số là cấu âm chứ không phải dựa trên blendshape, retarget sang danh tính mới là hiệu chỉnh Ánh xạ B theo từng rig, không phải huấn luyện lại mô hình thượng nguồn. Bộ mã hóa âm thanh, bộ ánh xạ IPA, và lớp động học đều độc lập với danh tính.

**Nhân vật phong cách hóa:** Khuôn mặt phong cách Pixar có tỷ lệ phóng đại nhưng cùng các bộ cấu âm trong cùng cấu hình. Ánh xạ C xử lý phong cách hóa. Cùng luồng ARTIC-48 điều khiển MetaHuman chụp ảnh thực tế có thể điều khiển con chó hoạt hình với xương và con cá với môi cao su.

**Thiếu giải phẫu:** Nhóm kênh phân cấp làm cho suy giảm duyên dáng trở nên tường minh. Không lưỡi → tắt S11–S18, phân phối lại trọng số sang môi/hàm qua Ánh xạ C. Không môi (mỏ) → tắt S04–S10, dựa vào hàm + lưỡi. Điều này là không thể làm sạch với biểu diễn phẳng 135 curve vì các curve được rối ràng về mặt giải phẫu.

---

## 8. So Sánh

| Thuộc tính | Viseme (~12–20) | ARKit 52 | MetaHuman 135 | **ARTIC-48** |
|---|---|---|---|---|
| Số chiều | ~15 | 52 | 135 | 48 |
| Độc lập ngôn ngữ | ❌ bộ theo từng ngôn ngữ | ⚠️ ngầm định | ⚠️ ngầm định | ✅ cấu âm |
| Tách biệt lời nói/cảm xúc | ❌ | ❌ | ❌ | ✅ theo tầng |
| Kênh có thể diễn giải | ✅ | ✅ | ✅ | ✅ |
| Nắm bắt đồng cấu âm | ❌ đích tĩnh | ⚠️ qua pha trộn | ⚠️ qua pha trộn | ✅ lớp động học |
| Biểu diễn lưỡi | ❌ | ❌ không có | ⚠️ một phần | ✅ 8 kênh |
| Có thể suy ra từ âm thanh | ⚠️ mất thông tin | ✅ | ❌ nhiều curve không theo dõi được | ✅ được thiết kế cho điều này |
| Phù hợp huấn luyện ML | ⚠️ ít thông tin | ✅ | ⚠️ đích rối ràng | ✅ |
| Chuyển giao đa ngôn ngữ | ❌ | ⚠️ | ⚠️ | ✅ |
| Hỗ trợ nhân vật phong cách hóa | ⚠️ xây lại mỗi phong cách | ⚠️ qua retarget | ⚠️ qua retarget | ✅ qua Ánh xạ C |
| Suy luận thời gian thực | ✅ | ✅ | ⚠️ nặng | ✅ |

**Hiệu quả thông tin:** ARTIC-48 mang mật độ thông tin trên mỗi kênh gấp khoảng 3 lần MetaHuman 135 vì các curve dư thừa (các cặp trái/phải của biến dạng má bên, v.v.) gộp vào các điều khiển cấp bộ cấu âm đơn. Theo kinh nghiệm, bạn có thể kỳ vọng 48 kênh cấu âm tái tạo ~95% phương sai liên quan đến lời nói trong 135 kênh với hiệu chỉnh đúng.

**Tổng quát hóa qua các ngôn ngữ:** Lợi thế lớn. Viseme cần bộ riêng cho mỗi họ ngôn ngữ. ARKit và MetaHuman *có thể* được điều khiển đa ngôn ngữ nhưng được huấn luyện trên dữ liệu chiếm ưu thế tiếng Anh, nên có bias ẩn (ví dụ biểu diễn lưỡi kém cho các ngôn ngữ lưỡi hiển thị nhiều). ARTIC-48 có các bộ cấu âm là công dân hạng nhất.

**Phù hợp cho huấn luyện ML:** Tách biệt là yếu tố quan trọng nhất. Mô hình được huấn luyện để dự đoán ARTIC-48 từ âm thanh không cần học rằng người nói đang mỉm cười — cảm xúc ở tầng riêng và có thể cung cấp hoặc lấy mẫu độc lập. Đây là cùng cái nhìn sau StyleGAN về tách biệt phong cách/nội dung và có cùng phần thưởng: mẫu tốt hơn, khả năng chỉnh sửa tốt hơn, chuyển giao tốt hơn.

---

## 9. Ví Dụ Thực Tế: /i/, /p/, /ʃ/

Vector đích Tầng S (bỏ qua giá trị zero; giá trị gần đúng, hiệu chỉnh dựa trên bó đặc trưng cấu âm IPA):

**/i/ — nguyên âm trước khép không tròn môi** (như trong "bee")

| Kênh | Giá trị | Lý do |
|---|---|---|
| S01 jaw_open | 0.15 | Mở nhẹ; nguyên âm cao |
| S04 lip_aperture | 0.20 | Khoảng cách môi hẹp |
| S06 lip_spreading | 0.65 | Kéo ngang mạnh |
| S11 tongue_height | 0.90 | Thân lưỡi cao |
| S12 tongue_backness | 0.10 | Vị trí lưỡi trước |
| S21 voicing | 1.00 | Hữu thanh |

**/p/ — phụ âm tắc hai môi vô thanh**

| Kênh | Giá trị | Lý do |
|---|---|---|
| S01 jaw_open | 0.05 | Gần đóng |
| S04 lip_aperture | 0.00 | Môi khép hoàn toàn |
| S07 lip_compression | 0.85 | Nén hai môi mạnh |
| S19 velum_lower | 0.00 | Phụ âm tắc miệng, màn hầu nâng |
| S21 voicing | 0.00 | Vô thanh |
| S22 aspiration | 0.50 | Bật hơi trong tiếng Anh |

*Sự chuyển tiếp* vào /p/ từ nguyên âm trước đó là điều quan trọng về mặt hoạt hình: lip_aperture giảm từ bất kỳ giá trị nào xuống 0 trong ~60ms, lip_compression tăng dần, jaw_open đóng một phần. Lớp động học trong §5 tạo ra quỹ đạo này tự động khi đích được đặt. Bước nhảy giải phóng (nén giảm nhanh, đỉnh bật hơi ngắn) cũng được xử lý bởi lớp động học với hành vi tắt dần tới hạn.

**/ʃ/ — âm xát sau răng huyệt vô thanh** (như trong "she")

| Kênh | Giá trị | Lý do |
|---|---|---|
| S01 jaw_open | 0.10 | Mở nhẹ |
| S04 lip_aperture | 0.25 | Khoảng cách nhỏ |
| S05 lip_rounding | 0.40 | Tròn môi nhẹ (/ʃ/ tiếng Anh tròn môi) |
| S13 tongue_tip_raise | 0.50 | Đầu lưỡi gần rặng răng |
| S15 tongue_groove | 0.85 | Rãnh mạnh cho âm xát |
| S16 tongue_dorsum_raise | 0.30 | Lưng lưỡi nâng nhẹ (sau răng huyệt) |
| S21 voicing | 0.00 | Vô thanh |

Hệ thống viseme sẽ nhóm /ʃ/ với /tʃ/, /ʒ/, /dʒ/ và gọi là "viseme G" — mất đi sự phân biệt tròn môi (/ʃ/ tiếng Anh tròn môi, /s/ thì không) và rãnh lưỡi (tạo hiệu ứng hõm má thấy được khi hoạt hình). ARTIC-48 bảo toàn chúng như các kênh độc lập.

---

## 10. Ưu, Nhược Điểm, Hạn Chế

**Ưu điểm**

- Nền tảng cấu âm cho hành vi độc lập ngôn ngữ thực sự, không chỉ là tuyên bố hỗ trợ đa ngôn ngữ.
- Tách biệt tầng làm cho chỉnh sửa, gỡ lỗi, và tạo sinh có điều kiện trở nên khả thi.
- Mô hình thời gian kết hợp động học + nơ-ron cho đồng cấu âm vật lý hợp lý với chi phí thấp.
- 48 kênh đủ nhỏ để suy luận nhanh và ma trận có thể diễn giải, đủ lớn để bảo toàn các phân biệt cấu âm mà viseme mất.
- Ánh xạ B là ma trận tuyến tính đã hiệu chỉnh bạn có thể đọc và kiểm tra — không có latent hộp đen.
- Suy giảm duyên dáng cho thiếu giải phẫu là cấu trúc, không phải giải pháp tạm.

**Nhược điểm**

- Các đích cấu âm cho mỗi âm vị đòi hỏi hiệu chỉnh ban đầu cẩn thận bởi các nhà ngữ âm học. Đây là chi phí một lần nhưng là chi phí thực sự.
- Ánh xạ B yêu cầu dữ liệu hiệu chỉnh theo từng danh tính (~30 phút biểu diễn ghép cặp), nhiều hơn ARKit nhưng ít hơn huấn luyện MetaHuman đầy đủ.
- Các kênh lưỡi khó xác nhận chỉ từ video ngoài — bạn cần dữ liệu siêu âm, MRI, hoặc articulography để có ground truth, rất tốn kém. Trên thực tế các tham số lưỡi phải được suy ra từ các tương quan âm học cộng chuyển động hàm/môi nhìn thấy.
- Ánh xạ C cho nhân vật phong cách hóa cao vẫn cần nhãn animator. Vật lý co giãn phim hoạt hình không phải cấu âm.

**Hạn chế**

- ARTIC-48 bao phủ lời nói phân đoạn và ngữ điệu tốt, cộng biểu cảm cảm xúc cốt lõi. Nó không bao phủ đầy đủ: cử chỉ ngôn ngoài ngữ (đảo mắt, nhướng mày dùng theo ngữ dụng), biểu cảm đặc thù văn hóa, hay ngữ pháp khuôn mặt ngôn ngữ ký hiệu (sử dụng mày/miệng theo cách ngôn ngữ học có hệ thống bên ngoài khung IPA).
- Ca hát và các chế độ phát âm kéo dài (giọng kẽ, giọng đầu, hát cổ họng) cần mở rộng Tầng S/P — có thể 4–6 kênh bổ sung thay vì tái tổ chức.
- Cử chỉ đồng hành lời nói (gật đầu theo trọng âm, nhướng mày theo tiêu điểm) được nắm bắt một phần bởi P01/P04 nhưng mô hình cử chỉ đầy đủ cần hệ thống cấp cơ thể riêng.
- Số 48 kênh là lựa chọn thiết kế, không phải bằng chứng. ARTIC-64 tương lai có thể thêm độ phân giải lưỡi tốt hơn (các kênh bên cho sự phân biệt /l/ so với /ɫ/, hoặc kênh riêng cho ngược so với vòm miệng). Kiến trúc hỗ trợ mở rộng mà không phá vỡ tương thích, vì cấu trúc tầng là điều quan trọng.

---

Lý do điều gì đó như thế này có thể trở thành tiêu chuẩn không phải vì nó khéo léo — mà vì lĩnh vực đang hội tụ về những ý tưởng này một cách độc lập. Nghiên cứu audio-to-face đang hướng tới các đích cấu âm (Audio2Face của NVIDIA sử dụng đặc trưng lưỡi; nghiên cứu gần đây về nghịch đảo cấu âm đang cung cấp cho các mô hình khuôn mặt). Các biểu diễn tách biệt là thực hành tiêu chuẩn trong mô hình tạo sinh. IPA đã là ngôn ngữ chung cho TTS đa ngôn ngữ. ARTIC-48 chỉ đơn giản là cam kết với cả ba cùng một lúc và phơi bày cấu trúc đủ rõ ràng để animator, kỹ sư ML, và rigger chia sẻ cùng một chỉ số.
