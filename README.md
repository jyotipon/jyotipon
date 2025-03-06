
DEFINE_UI_PARAMS(saturation, "Saturation", DCTLUI_SLIDER_FLOAT, 0.7, 0.0, 2.0, 0.01)
DEFINE_UI_PARAMS(contrast, "Contrast", DCTLUI_SLIDER_FLOAT, 1.0, 0.5, 2.0, 0.01)
DEFINE_UI_PARAMS(shadow_tint, "Shadow Tint", DCTLUI_SLIDER_FLOAT, 0.2, 0.0, 1.0, 0.01)
DEFINE_UI_PARAMS(highlight_warmth, "Highlight Warmth", DCTLUI_SLIDER_FLOAT, 0.2, 0.0, 1.0, 0.01)
DEFINE_UI_PARAMS(black_crush, "Black Crush", DCTLUI_SLIDER_FLOAT, 0.05, 0.0, 0.3, 0.01)
DEFINE_UI_PARAMS(halation, "Halation", DCTLUI_SLIDER_FLOAT, 0.1, 0.0, 0.5, 0.01)
DEFINE_UI_PARAMS(gamma, "Gamma", DCTLUI_SLIDER_FLOAT, 1.0, 0.5, 2.5, 0.01)
DEFINE_UI_PARAMS(vibrance, "Vibrance", DCTLUI_SLIDER_FLOAT, 0.5, 0.0, 1.0, 0.01)
DEFINE_UI_PARAMS(hue_shift, "Hue Shift", DCTLUI_SLIDER_FLOAT, 0.0, -0.5, 0.5, 0.01)
DEFINE_UI_PARAMS(mid_tone_balance, "Mid Tone Balance", DCTLUI_SLIDER_FLOAT, 0.0, -1.0, 1.0, 0.01)
DEFINE_UI_PARAMS(film_grain, "Film Grain", DCTLUI_SLIDER_FLOAT, 0.0, 0.0, 1.0, 0.01)
DEFINE_UI_PARAMS(bloom, "Bloom", DCTLUI_SLIDER_FLOAT, 0.1, 0.0, 1.0, 0.01)
DEFINE_UI_PARAMS(glow_intensity, "Glow Intensity", DCTLUI_SLIDER_FLOAT, 0.1, 0.0, 1.0, 0.01)
DEFINE_UI_PARAMS(toe_strength, "Toe Strength", DCTLUI_SLIDER_FLOAT, 0.1, 0.0, 1.0, 0.01)
DEFINE_UI_PARAMS(shoulder_strength, "Shoulder Strength", DCTLUI_SLIDER_FLOAT, 0.1, 0.0, 1.0, 0.01)
DEFINE_UI_PARAMS(highlight_recovery, "Highlight Recovery", DCTLUI_SLIDER_FLOAT, 0.3, 0.0, 1.0, 0.01)
DEFINE_UI_PARAMS(log_saturation, "Log Saturation", DCTLUI_SLIDER_FLOAT, 1.0, 0.0, 2.0, 0.01)

// Main transform function
__DEVICE__ float3 transform(
    int p_Width, int p_Height, int p_X, int p_Y,
    float p_R, float p_G, float p_B)
{
    float3 color = make_float3(p_R, p_G, p_B);
    color = clamp(color, 0.0, 1.0);
    color = pow(color, make_float3(2.2));

    float highlightLuma = dot(color, make_float3(0.299, 0.587, 0.114));
    color = mix(color, pow(color, make_float3(0.85)), highlight_recovery * smoothstep(0.7, 1.0, highlightLuma));

    float3 shadowColor = make_float3(0.0, 0.2, 0.3);
    float shadowLuma = dot(color, make_float3(0.299, 0.587, 0.114));
    color = mix(color, shadowColor, (1.0 - shadowLuma) * shadow_tint);

    float3 warmTone = make_float3(1.0, 0.85, 0.7);
    color = mix(color, warmTone, smoothstep(0.5, 1.0, highlightLuma) * highlight_warmth);

    float midGrey = 0.18;
    color = pow((color / midGrey), contrast) * midGrey;
    color = max((color - black_crush) / (1.0 - black_crush), 0.0);

    float halationEffect = smoothstep(0.6, 1.0, highlightLuma) * halation;
    color += halationEffect * make_float3(1.0, 0.6, 0.4) * 0.2;

    color += bloom * pow(highlightLuma, 2.0);

    float logColor = log2(1.0 + dot(color, make_float3(0.2126, 0.7152, 0.0722)));
    color = mix(make_float3(logColor), color, log_saturation);

    float3 avgColor = (color + dot(color, make_float3(0.2126, 0.7152, 0.0722))) * 0.5;
    color = mix(avgColor, color, vibrance);

    float3 shiftedColor = make_float3(
        color.r + hue_shift * (color.g - color.b),
        color.g + hue_shift * (color.b - color.r),
        color.b + hue_shift * (color.r - color.g)
    );
    color = mix(color, shiftedColor, fabs(hue_shift));

    float midToneAdjust = smoothstep(0.3, 0.7, shadowLuma);
    color = mix(color, color * make_float3(1.0 + mid_tone_balance, 1.0, 1.0 - mid_tone_balance), midToneAdjust);

    color += film_grain * (fract(sin(dot(color, make_float3(12.9898, 78.233, 45.164))) * 43758.5453) - 0.5) * (1.0 - highlightLuma);
    color += glow_intensity * pow(highlightLuma, 3.0);

    color = pow(color, make_float3(1.0 / (1.0 + toe_strength)));
    color = 1.0 - pow(1.0 - color, make_float3(1.0 / (1.0 + shoulder_strength)));

    color = pow(color, make_float3(1.0 / gamma));

    return clamp(color, 0.0, 1.0);
}
