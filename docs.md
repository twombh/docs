# 다른 앱 위에 그리기
## dynamic.service.DragHandler
package com.example.dynamic.service // DragHandler 클래스가 위치한 패키지 지정

import android.content.Context // 시스템 서비스 및 리소스 접근을 위한 Context
import android.content.Intent // 액티비티 이동을 위한 Intent
import android.content.res.Resources // 해상도 등 디바이스 리소스 접근
import android.graphics.Color // 색상 지정
import android.graphics.drawable.GradientDrawable // 배경 drawable 설정
import android.view.MotionEvent // 터치 이벤트 추적
import android.view.View // View 객체
import android.view.WindowManager // 오버레이 뷰의 위치 제어
import com.example.dynamic.MainActivity // 클릭 시 돌아갈 메인 액티비티
import kotlin.math.abs // 거리 계산 시 음수 제거용 함수

class DragHandler( // DragHandler는 오버레이 뷰의 드래그 및 클릭 처리를 담당
    private val windowManager: WindowManager, // 뷰 위치 업데이트에 사용
    private val resources: Resources, // 화면 크기 등 리소스 접근
    private val context: Context // 인텐트 시작 등 앱 컨텍스트
) {

    // 드래그 동작을 위한 좌표 상태 변수들
    private var initialX = 0 // 드래그 시작 시의 오버레이 X 좌표
    private var initialY = 0 // 드래그 시작 시의 오버레이 Y 좌표
    private var initialTouchX = 0f // 터치 시작 X 좌표 (화면 기준)
    private var initialTouchY = 0f // 터치 시작 Y 좌표 (화면 기준)
    private var isDragging = false // 현재 드래그 중인지 여부
    private val dragThreshold = 10f // 드래그로 인식할 최소 이동 거리 (10px)

    // 터치 피드백 처리 전 배경 저장용 변수
    private var originalBackground: GradientDrawable? = null // 원래 배경을 저장

    /**
     * 외부에서 호출되는 터치 이벤트 처리 메서드
     */
    fun handleTouchEvent(
        event: MotionEvent, // 현재 터치 이벤트
        overlayView: View, // 드래그 대상 오버레이 뷰
        params: WindowManager.LayoutParams // 오버레이 뷰의 현재 레이아웃 파라미터
    ): Boolean {
        when (event.action) {
            MotionEvent.ACTION_DOWN -> {
                return handleTouchDown(event, overlayView, params) // 터치 시작 처리
            }
            MotionEvent.ACTION_MOVE -> {
                return handleTouchMove(event, overlayView, params) // 드래그 이동 처리
            }
            MotionEvent.ACTION_UP -> {
                return handleTouchUp(overlayView) // 터치 끝 처리
            }
            MotionEvent.ACTION_CANCEL -> {
                return handleTouchCancel(overlayView) // 이벤트 취소 처리
            }
        }
        return false // 처리하지 않은 이벤트의 경우 false 반환
    }

    /**
     * 터치가 시작될 때 호출
     */
    private fun handleTouchDown(
        event: MotionEvent,
        overlayView: View,
        params: WindowManager.LayoutParams
    ): Boolean {
        isDragging = false // 아직 드래그로 판단하지 않음
        initialX = params.x // 시작 시점의 뷰 X 위치 저장
        initialY = params.y // 시작 시점의 뷰 Y 위치 저장
        initialTouchX = event.rawX // 터치 시작 X 좌표 (절대 좌표)
        initialTouchY = event.rawY // 터치 시작 Y 좌표 (절대 좌표)

        saveOriginalBackground(overlayView) // 원래 배경 저장
        applyTouchFeedback(overlayView) // 피드백용 배경 적용
        return true
    }

    /**
     * 터치 이동 시 호출되는 로직
     */
    private fun handleTouchMove(
        event: MotionEvent,
        overlayView: View,
        params: WindowManager.LayoutParams
    ): Boolean {
        val deltaX = event.rawX - initialTouchX // 움직인 X 거리 계산
        val deltaY = event.rawY - initialTouchY // 움직인 Y 거리 계산

        // 드래그 시작 조건 충족 여부 판단
        if (!isDragging && (abs(deltaX) > dragThreshold || abs(deltaY) > dragThreshold)) {
            isDragging = true // 드래그 상태 진입
        }

        if (isDragging) {
            val newX = initialX + deltaX.toInt() // 새 X 위치 계산
            val newY = initialY + deltaY.toInt() // 새 Y 위치 계산

            val screenBounds = getScreenBounds() // 화면 해상도 가져오기
            val clampedX = newX.coerceIn(0, screenBounds.first - params.width) // 화면 경계 제한
            val clampedY = newY.coerceIn(0, screenBounds.second - params.height)

            updateOverlayPosition(overlayView, params, clampedX, clampedY) // 위치 업데이트
        }
        return true
    }

    /**
     * 손가락 뗐을 때 호출
     */
    private fun handleTouchUp(overlayView: View): Boolean {
        restoreOriginalBackground(overlayView) // 배경 원복

        if (!isDragging) {
            performClickAnimation(overlayView) // 클릭 애니메이션 실행
            navigateToMainApp() // 앱의 MainActivity로 이동
        }

        isDragging = false // 드래그 상태 초기화
        return true
    }

    /**
     * 터치 이벤트가 취소됐을 때 호출
     */
    private fun handleTouchCancel(overlayView: View): Boolean {
        restoreOriginalBackground(overlayView) // 배경 복원
        isDragging = false // 상태 초기화
        return true
    }

    /**
     * 오버레이 뷰 위치 업데이트
     */
    private fun updateOverlayPosition(
        overlayView: View,
        params: WindowManager.LayoutParams,
        x: Int,
        y: Int
    ) {
        params.x = x // 새로운 X 좌표 적용
        params.y = y // 새로운 Y 좌표 적용
        windowManager.updateViewLayout(overlayView, params) // 오버레이 위치 실제 반영
    }

    /**
     * 화면 해상도 (width, height) 반환
     */
    private fun getScreenBounds(): Pair<Int, Int> {
        val displayMetrics = resources.displayMetrics // 디바이스 해상도 정보
        return Pair(displayMetrics.widthPixels, displayMetrics.heightPixels) // 픽셀 단위 반환
    }

    /**
     * 현재 배경을 복사하여 저장
     */
    private fun saveOriginalBackground(overlayView: View) {
        originalBackground =
            (overlayView.background as? GradientDrawable)?.constantState?.newDrawable() as? GradientDrawable
    }

    /**
     * 터치 시 회색 배경 피드백 적용
     */
    private fun applyTouchFeedback(overlayView: View) {
        val touchBackground = GradientDrawable().apply {
            shape = GradientDrawable.RECTANGLE // 사각형 배경
            setColor(Color.parseColor("#F0F0F0")) // 연한 회색
            cornerRadius = 30f * resources.displayMetrics.density // 30dp → px
        }
        overlayView.background = touchBackground // 배경 적용
    }

    /**
     * 원래 배경으로 되돌리기
     */
    private fun restoreOriginalBackground(overlayView: View) {
        originalBackground?.let { background ->
            overlayView.background = background // 저장된 배경이 있다면 복원
        } ?: run {
            val defaultBackground = GradientDrawable().apply {
                shape = GradientDrawable.RECTANGLE
                setColor(Color.WHITE) // 기본 흰색 배경
                cornerRadius = 30f * resources.displayMetrics.density
            }
            overlayView.background = defaultBackground // 없으면 기본 배경
        }
    }

    /**
     * 클릭 시 뷰에 애니메이션 효과 부여
     */
    private fun performClickAnimation(overlayView: View) {
        overlayView.animate()
            .scaleX(1.05f)
            .scaleY(1.05f)
            .setDuration(100)
            .withEndAction {
                overlayView.animate()
                    .scaleX(1f)
                    .scaleY(1f)
                    .setDuration(100)
                    .start()
            }
            .start()
    }

    /**
     * 메인 액티비티로 이동
     */
    private fun navigateToMainApp() {
        try {
            val intent = Intent(context, MainActivity::class.java).apply {
                flags = Intent.FLAG_ACTIVITY_NEW_TASK or // 새 태스크로 시작
                        Intent.FLAG_ACTIVITY_CLEAR_TOP or // 기존 액티비티 제거
                        Intent.FLAG_ACTIVITY_SINGLE_TOP // 이미 실행 중이면 재사용
            }
            context.startActivity(intent)
        } catch (e: Exception) {
            e.printStackTrace() // 예외 발생 시 로그 출력
        }
    }

    /**
     * 현재 드래그 상태 반환
     */
    fun isDragging(): Boolean = isDragging
}

----
package com.example.dynamic.service // OverlayManager 클래스가 위치한 패키지

import android.content.Context // 애플리케이션 또는 서비스 컨텍스트 제공
import android.graphics.PixelFormat // 픽셀 포맷 설정 (투명도 등)
import android.graphics.drawable.GradientDrawable // 배경 Drawable 설정용 클래스
import android.os.Build // SDK 버전 체크용
import android.util.DisplayMetrics // 화면 해상도 및 밀도 정보
import android.util.Log // 로그 출력
import android.util.TypedValue // dp, sp 등을 px로 변환
import android.view.Gravity // 뷰의 기본 정렬 방향 설정
import android.view.LayoutInflater // XML 레이아웃을 View로 변환
import android.view.View // 뷰 객체
import android.view.WindowManager // 시스템 오버레이를 관리하는 주요 클래스
import android.widget.LinearLayout // 오버레이 컨테이너 레이아웃
import android.widget.TextView // 텍스트 뷰
import com.example.dynamic.R // 리소스 접근용

class OverlayManager( // 오버레이 뷰를 생성, 표시, 제거하는 클래스
    private val context: Context, // 컨텍스트 (서비스 또는 앱)
    private val windowManager: WindowManager // 시스템 창 관리자
) {

    private var overlayView: View? = null // 현재 표시 중인 오버레이 뷰
    private var layoutParams: WindowManager.LayoutParams? = null // 오버레이의 레이아웃 속성

    // 반응형 크기 처리를 위한 화면 정보와 계산된 크기
    private val screenMetrics = getScreenMetrics() // 현재 디바이스 해상도 정보
    private val responsiveSizes = calculateResponsiveSizes() // 해상도 기반 동적 크기 설정

    companion object {
        private const val TAG = "OverlayManager" // 로그 태그

        // 기준 화면 크기 (dp 단위, CSS-like 기준)
        private const val REFERENCE_SCREEN_WIDTH = 412f
        private const val REFERENCE_SCREEN_HEIGHT = 892f

        // 기준 오버레이 UI 요소 크기
        private const val REFERENCE_OVERLAY_WIDTH = 276f
        private const val REFERENCE_OVERLAY_HEIGHT = 100f
        private const val REFERENCE_BORDER_RADIUS = 30f
        private const val REFERENCE_FONT_SIZE_LARGE = 22f
        private const val REFERENCE_FONT_SIZE_SMALL = 16f
    }

    // 화면 정보 획득 (서비스에서는 WindowManager로만 접근 가능)
    private fun getScreenMetrics(): DisplayMetrics {
        val metrics = DisplayMetrics()
        @Suppress("DEPRECATION") // 오래된 API 사용 허용
        windowManager.defaultDisplay.getRealMetrics(metrics) // 실제 화면 크기 저장
        return metrics
    }

    // 해상도에 따라 비율을 계산하고 적절한 크기 반환
    private fun calculateResponsiveSizes(): ResponsiveSizes {
        val screenWidthPx = screenMetrics.widthPixels.toFloat()
        val screenHeightPx = screenMetrics.heightPixels.toFloat()
        val density = screenMetrics.density

        // 화면 크기를 dp 단위로 변환
        val screenWidthDp = screenWidthPx / density
        val screenHeightDp = screenHeightPx / density

        Log.d(TAG, "화면 크기: ${screenWidthDp}dp x ${screenHeightDp}dp (밀도: $density)")

        // 기준 화면 대비 비율 계산
        val widthRatio = screenWidthDp / REFERENCE_SCREEN_WIDTH
        val heightRatio = screenHeightDp / REFERENCE_SCREEN_HEIGHT

        // 더 작은 비율로 스케일링 (화면 넘침 방지)
        val scaleFactor = minOf(widthRatio, heightRatio)

        Log.d(TAG, "스케일 팩터: $scaleFactor (가로비: $widthRatio, 세로비: $heightRatio)")

        return ResponsiveSizes(
            overlayWidth = (REFERENCE_OVERLAY_WIDTH * scaleFactor).toInt(),
            overlayHeight = (REFERENCE_OVERLAY_HEIGHT * scaleFactor).toInt(),
            borderRadius = REFERENCE_BORDER_RADIUS * scaleFactor,
            fontSizeLarge = REFERENCE_FONT_SIZE_LARGE * scaleFactor,
            fontSizeSmall = REFERENCE_FONT_SIZE_SMALL * scaleFactor,
            logoSize = (50f * scaleFactor).toInt(),
            gap16dp = (16f * scaleFactor).toInt(),
            gap12dp = (12f * scaleFactor).toInt(),
            gap4dp = (4f * scaleFactor).toInt(),
            gap2dp = (2f * scaleFactor).toInt(),
            padding24dp = (24f * scaleFactor).toInt()
        )
    }

    // 반응형 크기들을 담는 데이터 클래스
    data class ResponsiveSizes(
        val overlayWidth: Int,
        val overlayHeight: Int,
        val borderRadius: Float,
        val fontSizeLarge: Float,
        val fontSizeSmall: Float,
        val logoSize: Int,
        val gap16dp: Int,
        val gap12dp: Int,
        val gap4dp: Int,
        val gap2dp: Int,
        val padding24dp: Int
    )

    // 오버레이 생성 및 뷰와 레이아웃 파라미터 반환
    fun createOverlay(onTouchListener: View.OnTouchListener): Pair<View, WindowManager.LayoutParams>? {
        try {
            val inflater = LayoutInflater.from(context)
            Log.d(TAG, "XML 레이아웃 inflate 시작: R.layout.overlay_layout")
            val view = inflater.inflate(R.layout.overlay_layout, null)
            Log.d(TAG, "XML 레이아웃 inflate 완료: ${view::class.simpleName}")

            applyResponsiveSizes(view) // 크기 반응형 적용
            setupBackgrounds(view) // 배경 적용
            val params = createLayoutParams() // 레이아웃 파라미터 생성
            view.setOnTouchListener(onTouchListener) // 터치 리스너 설정
            windowManager.addView(view, params) // 오버레이 화면에 추가

            overlayView = view
            layoutParams = params

            Log.d(TAG, "오버레이 생성 완료 (반응형 크기: ${responsiveSizes.overlayWidth}x${responsiveSizes.overlayHeight}dp)")
            return Pair(view, params)
        } catch (e: Exception) {
            Log.e(TAG, "오버레이 생성 실패: ${e.message}", e)
            return null
        }
    }

    // View에 반응형 크기 적용
    private fun applyResponsiveSizes(view: View) {
        val container = view.findViewById<LinearLayout>(R.id.overlayContainer)
        container?.let { cont ->
            val paddingPx = dpToPx(responsiveSizes.padding24dp.toFloat()).toInt()
            cont.setPadding(paddingPx, 0, paddingPx, 0)
            cont.minimumWidth = dpToPx(responsiveSizes.overlayWidth.toFloat()).toInt()
            cont.minimumHeight = dpToPx(responsiveSizes.overlayHeight.toFloat()).toInt()
            Log.d(TAG, "컨테이너 설정: 최소크기 ${responsiveSizes.overlayWidth}x${responsiveSizes.overlayHeight}dp, 패딩 ${responsiveSizes.padding24dp}dp")
        }

        val logoView = view.findViewById<View>(R.id.logoView)
        logoView?.let { logo ->
            val logoParams = logo.layoutParams as? LinearLayout.LayoutParams ?: LinearLayout.LayoutParams(
                dpToPx(responsiveSizes.logoSize.toFloat()).toInt(),
                dpToPx(responsiveSizes.logoSize.toFloat()).toInt()
            )
            logoParams.width = dpToPx(responsiveSizes.logoSize.toFloat()).toInt()
            logoParams.height = dpToPx(responsiveSizes.logoSize.toFloat()).toInt()
            logoParams.setMargins(0, 0, dpToPx(responsiveSizes.gap16dp.toFloat()).toInt(), 0)
            logo.layoutParams = logoParams
            Log.d(TAG, "로고 설정: ${responsiveSizes.logoSize}x${responsiveSizes.logoSize}dp")
        }

        adjustSpaces(view) // 뷰 사이 여백 조정
        adjustTextSizes(view) // 텍스트 크기 조정
        Log.d(TAG, "반응형 크기 적용 완료: ${responsiveSizes}")
    }

    // 공간 여백 조정
    private fun adjustSpaces(view: View) {
        view.findViewById<View>(R.id.topSpace)?.let { space ->
            val params = space.layoutParams
            params.height = dpToPx(responsiveSizes.gap12dp.toFloat() / 2).toInt()
            space.layoutParams = params
        }

        view.findViewById<View>(R.id.middleSpace)?.let { space ->
            val params = space.layoutParams
            params.width = dpToPx(responsiveSizes.gap4dp.toFloat()).toInt()
            space.layoutParams = params
        }

        view.findViewById<View>(R.id.bottomSpace)?.let { space ->
            val params = space.layoutParams
            params.height = dpToPx(responsiveSizes.gap2dp.toFloat()).toInt()
            space.layoutParams = params
        }

        view.findViewById<View>(R.id.bottomSpaceMain)?.let { space ->
            val params = space.layoutParams
            params.height = dpToPx(responsiveSizes.gap12dp.toFloat() / 2).toInt()
            space.layoutParams = params
        }

        Log.d(TAG, "간격 조정 완료: gap12=${responsiveSizes.gap12dp}dp, gap4=${responsiveSizes.gap4dp}dp, gap2=${responsiveSizes.gap2dp}dp")
    }

    // 텍스트 크기 조정
    private fun adjustTextSizes(view: View) {
        view.findViewById<TextView>(R.id.ddareungiText)?.let { textView ->
            textView.setTextSize(TypedValue.COMPLEX_UNIT_SP, responsiveSizes.fontSizeLarge)
            Log.d(TAG, "따릉이 텍스트 크기: ${responsiveSizes.fontSizeLarge}sp")
        }

        view.findViewById<TextView>(R.id.daeyeojungText)?.let { textView ->
            textView.setTextSize(TypedValue.COMPLEX_UNIT_SP, responsiveSizes.fontSizeLarge)
            Log.d(TAG, "대여중 텍스트 크기: ${responsiveSizes.fontSizeLarge}sp")
        }

        view.findViewById<TextView>(R.id.subtitleText)?.let { textView ->
            textView.setTextSize(TypedValue.COMPLEX_UNIT_SP, responsiveSizes.fontSizeSmall)
            Log.d(TAG, "부제목 텍스트 크기: ${responsiveSizes.fontSizeSmall}sp")
        }
    }

    // dp를 px로 변환 (밀도 고려)
    private fun dpToPx(dp: Float): Float {
        return TypedValue.applyDimension(
            TypedValue.COMPLEX_UNIT_DIP,
            dp,
            context.resources.displayMetrics
        )
    }

    // 배경 적용
    private fun setupBackgrounds(view: View) {
        val container = view.findViewById<View>(R.id.overlayContainer)
        if (container != null) {
            container.background = createRoundedBackground()
            Log.d(TAG, "컨테이너 배경 설정 완료")
        } else {
            Log.e(TAG, "overlayContainer ID 찾을 수 없음!")
        }

        val logoView = view.findViewById<View>(R.id.logoView)
        if (logoView != null) {
            logoView.background = createLogoBackground()
            Log.d(TAG, "로고 배경 설정 완료")
        } else {
            Log.e(TAG, "logoView ID 찾을 수 없음!")
        }
    }

    // 흰색 둥근 사각형 배경 생성
    private fun createRoundedBackground(): GradientDrawable {
        return GradientDrawable().apply {
            shape = GradientDrawable.RECTANGLE
            setColor(android.graphics.Color.WHITE)
            cornerRadius = dpToPx(responsiveSizes.borderRadius)
        }
    }

    // 초록색 원형 배경 생성
    private fun createLogoBackground(): GradientDrawable {
        return GradientDrawable().apply {
            shape = GradientDrawable.OVAL
            setColor(android.graphics.Color.parseColor("#4CAF50"))
        }
    }

    // 오버레이 제거
    fun removeOverlay() {
        overlayView?.let { view ->
            try {
                windowManager.removeView(view)
                Log.d(TAG, "오버레이 제거 완료")
            } catch (e: Exception) {
                Log.e(TAG, "오버레이 제거 실패: ${e.message}", e)
            }
        }
        overlayView = null
        layoutParams = null
    }

    // 레이아웃 파라미터 생성
    private fun createLayoutParams(): WindowManager.LayoutParams {
        val params = WindowManager.LayoutParams(
            WindowManager.LayoutParams.WRAP_CONTENT,
            WindowManager.LayoutParams.WRAP_CONTENT,
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY
            } else {
                @Suppress("DEPRECATION")
                WindowManager.LayoutParams.TYPE_PHONE
            },
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or
                    WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL or
                    WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH,
            PixelFormat.TRANSLUCENT
        )
        params.gravity = Gravity.TOP or Gravity.START // 왼쪽 위 기준으로 배치
        params.x = 100 // 초기 x 위치
        params.y = 100 // 초기 y 위치
        return params
    }

    // 오버레이 뷰 반환
    fun getOverlayView(): View? = overlayView

    // 레이아웃 파라미터 반환
    fun getLayoutParams(): WindowManager.LayoutParams? = layoutParams

    // 오버레이가 이미 생성되었는지 여부 반환
    fun isOverlayCreated(): Boolean = overlayView != null
}
