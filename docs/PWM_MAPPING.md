# Pemetaan PWM SvayamV3 (LQFP176) — 13 Nov 2026

Urutan timer diubah supaya fanout dari MCU ke rel servo tanpa persilangan (`pwm_route_preview.py`: 0).
Label "Channel N" di skematik tidak diubah; yang berubah hanya net timer pada resistor seri.

| Channel (label konektor) | Konektor | R seri | Timer | Pin MCU |
|---|---|---|---|---|
| Channel 1 | J19 | R68 | TIM2_CH1 | 40 |
| Channel 2 | J23 | R72 | TIM2_CH3 | 42 |
| Channel 3 | J27 | R76 | TIM3_CH3 | 56 |
| Channel 4 | J31 | R80 | TIM5_CH1 | 87 |
| Channel 5 | J20 | R69 | TIM4_CH3 | 104 |
| Channel 6 | J24 | R73 | TIM8_CH2 | 116 |
| Channel 7 | J28 | R77 | TIM1_CH2 | 120 |
| Channel 8 | J32 | R81 | TIM8_CH4 | 133 |
| Channel 9 | J21 | R70 | TIM2_CH2 | 41 |
| Channel 10 | J25 | R74 | TIM3_CH2 | 53 |
| Channel 11 | J29 | R78 | TIM3_CH4 | 57 |
| Channel 12 | J33 | R82 | TIM5_CH2 | 88 |
| Channel 13 | J22 | R71 | TIM4_CH2 | 101 |
| Channel 14 | J26 | R75 | TIM8_CH1 | 115 |
| Channel 15 | J30 | R79 | TIM1_CH1 | 119 |
| Channel 16 | J34 | R83 | TIM1_CH3 | 121 |

Catatan hwdef: nomor channel per timer tidak lagi berurutan (mis. TIM2 = Channel 1, 9, 2;
TIM3 = 10, 3, 11; TIM5 = 4, 12; TIM4 = 13, 5; TIM8 = 14, 6, 8; TIM1 = 15, 7, 16).
Grup timer yang berbagi frekuensi PWM perlu disesuaikan di firmware.
