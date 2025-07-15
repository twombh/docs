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

## dynamic.service.OverlayManager
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

## dynamic.service.OverlayManager2
package com.example.dynamic.service // 이 클래스가 포함된 패키지를 지정합니다.

import android.content.Context // 애플리케이션/서비스 전역 상태를 제공하는 컨텍스트
import android.graphics.PixelFormat // 윈도우 투명도 설정을 위한 픽셀 포맷 상수
import android.graphics.drawable.GradientDrawable // 코드로 배경을 그릴 수 있게 해주는 Drawable 클래스
import android.os.Build // 현재 안드로이드 SDK 버전을 체크하기 위한 클래스
import android.util.DisplayMetrics // 디바이스의 해상도 및 밀도 정보를 담는 클래스
import android.util.Log // 로그 출력용 유틸
import android.util.TypedValue // dp나 sp를 px로 변환할 수 있게 해주는 클래스
import android.view.Gravity // 뷰의 위치 기준점 설정 (예: 왼쪽 상단)
import android.view.LayoutInflater // XML 레이아웃을 실제 View 객체로 바꿔주는 클래스
import android.view.View // 모든 UI 요소의 기본 클래스
import android.view.WindowManager // 시스템 오버레이를 관리하기 위한 클래스
import android.widget.LinearLayout // 오버레이 내부 레이아웃 컨테이너
import android.widget.TextView // 텍스트를 표시하는 뷰
import com.example.dynamic.R // 리소스 ID에 접근하기 위한 import

class OverlayManager(
    private val context: Context, // Context: 뷰 생성, 리소스 접근, 인플레이터 등 전반에 필요
    private val windowManager: WindowManager // WindowManager: 오버레이를 추가/제거할 때 사용
) {

    private var overlayView: View? = null // 실제 화면에 추가된 오버레이 뷰 (nullable)
    private var layoutParams: WindowManager.LayoutParams? = null // 오버레이의 배치 속성 (위치, 크기, 투명도 등)

    // 현재 기기의 해상도 및 밀도 정보
    private val screenMetrics = getScreenMetrics()

    // 위 해상도를 기반으로 계산된 반응형 크기들
    private val responsiveSizes = calculateResponsiveSizes()

    companion object {
        private const val TAG = "OverlayManager" // 로그 태그

        // 기준 화면 크기 (dp 기준). 다른 기기에서 비율 계산 시 기준이 됨
        private const val REFERENCE_SCREEN_WIDTH = 412f
        private const val REFERENCE_SCREEN_HEIGHT = 892f

        // 기준 오버레이 UI 요소 크기들 (반응형 계산의 기준값)
        private const val REFERENCE_OVERLAY_WIDTH = 276f
        private const val REFERENCE_OVERLAY_HEIGHT = 100f
        private const val REFERENCE_BORDER_RADIUS = 30f
        private const val REFERENCE_FONT_SIZE_LARGE = 22f
        private const val REFERENCE_FONT_SIZE_SMALL = 16f
    }

    // 현재 기기의 화면 해상도 및 밀도 정보 추출
    private fun getScreenMetrics(): DisplayMetrics {
        val metrics = DisplayMetrics()
        // Service에서는 Activity처럼 디스플레이 접근이 불가능하므로 WindowManager에서 가져와야 함
        @Suppress("DEPRECATION") // getRealMetrics는 deprecated 되었지만 여전히 Service에서 유효함
        windowManager.defaultDisplay.getRealMetrics(metrics) // 화면 전체 크기(px) 및 밀도 정보를 metrics에 담음
        return metrics
    }

    // 현재 화면 비율을 기준으로 오버레이의 크기를 동적으로 조정
    private fun calculateResponsiveSizes(): ResponsiveSizes {
        val screenWidthPx = screenMetrics.widthPixels.toFloat() // 화면 너비(px)
        val screenHeightPx = screenMetrics.heightPixels.toFloat() // 화면 높이(px)
        val density = screenMetrics.density // 밀도 (예: mdpi=1.0, xhdpi=2.0)

        // 화면 크기를 dp로 환산 (px / density = dp)
        val screenWidthDp = screenWidthPx / density
        val screenHeightDp = screenHeightPx / density

        Log.d(TAG, "화면 크기: ${screenWidthDp}dp x ${screenHeightDp}dp (밀도: $density)")

        // 기준 화면 크기 대비 현재 화면의 스케일 비율 계산
        val widthRatio = screenWidthDp / REFERENCE_SCREEN_WIDTH
        val heightRatio = screenHeightDp / REFERENCE_SCREEN_HEIGHT

        // 가로/세로 중 더 작은 비율을 선택하여 오버레이가 화면 밖으로 나가지 않게 함
        val scaleFactor = minOf(widthRatio, heightRatio)

        Log.d(TAG, "스케일 팩터: $scaleFactor (가로비: $widthRatio, 세로비: $heightRatio)")

        // 반응형 크기 값을 계산해서 데이터 클래스에 저장
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

    // 반응형 크기를 담는 데이터 클래스 정의
    data class ResponsiveSizes(
        val overlayWidth: Int, // 오버레이 너비
        val overlayHeight: Int, // 오버레이 높이
        val borderRadius: Float, // 모서리 둥글기
        val fontSizeLarge: Float, // 큰 텍스트 크기
        val fontSizeSmall: Float, // 작은 텍스트 크기
        val logoSize: Int, // 로고 크기 (정사각형)
        val gap16dp: Int, // 내부 여백 (16dp 기준)
        val gap12dp: Int,
        val gap4dp: Int,
        val gap2dp: Int,
        val padding24dp: Int // 외부 패딩
    )

    // 오버레이 뷰를 생성하고 화면에 추가
    fun createOverlay(onTouchListener: View.OnTouchListener): Pair<View, WindowManager.LayoutParams>? {
        try {
            val inflater = LayoutInflater.from(context) // 레이아웃 인플레이터 생성
            Log.d(TAG, "XML 레이아웃 inflate 시작: R.layout.overlay_layout")
            val view = inflater.inflate(R.layout.overlay_layout, null) // XML을 실제 View로 변환
            Log.d(TAG, "XML 레이아웃 inflate 완료: ${view::class.simpleName}")

            applyResponsiveSizes(view) // 뷰에 크기 및 텍스트 사이즈 적용
            setupBackgrounds(view) // 배경 설정 (모서리 둥글기 등)
            val params = createLayoutParams() // 오버레이 배치 속성 생성
            view.setOnTouchListener(onTouchListener) // 터치 리스너 연결
            windowManager.addView(view, params) // 화면에 뷰 추가

            // 내부 상태 저장
            overlayView = view
            layoutParams = params

            Log.d(TAG, "오버레이 생성 완료 (반응형 크기: ${responsiveSizes.overlayWidth}x${responsiveSizes.overlayHeight}dp)")
            return Pair(view, params)
        } catch (e: Exception) {
            Log.e(TAG, "오버레이 생성 실패: ${e.message}", e)
            return null
        }
    }

    // 오버레이 뷰 내부에 반응형 크기를 적용
    private fun applyResponsiveSizes(view: View) {
        // 1. 컨테이너 최소 크기 및 패딩 설정
        val container = view.findViewById<LinearLayout>(R.id.overlayContainer)
        container?.let { cont ->
            val paddingPx = dpToPx(responsiveSizes.padding24dp.toFloat()).toInt()
            cont.setPadding(paddingPx, 0, paddingPx, 0) // 좌우 패딩 적용
            cont.minimumWidth = dpToPx(responsiveSizes.overlayWidth.toFloat()).toInt()
            cont.minimumHeight = dpToPx(responsiveSizes.overlayHeight.toFloat()).toInt()

            Log.d(TAG, "컨테이너 설정: 최소크기 ${responsiveSizes.overlayWidth}x${responsiveSizes.overlayHeight}dp, 패딩 ${responsiveSizes.padding24dp}dp")
        }

        // 2. 로고 크기 설정
        val logoView = view.findViewById<View>(R.id.logoView)
        logoView?.let { logo ->
            val logoParams = logo.layoutParams as? LinearLayout.LayoutParams
                ?: LinearLayout.LayoutParams(
                    dpToPx(responsiveSizes.logoSize.toFloat()).toInt(),
                    dpToPx(responsiveSizes.logoSize.toFloat()).toInt()
                )
            logoParams.width = dpToPx(responsiveSizes.logoSize.toFloat()).toInt()
            logoParams.height = dpToPx(responsiveSizes.logoSize.toFloat()).toInt()
            logoParams.setMargins(0, 0, dpToPx(responsiveSizes.gap16dp.toFloat()).toInt(), 0)
            logo.layoutParams = logoParams

            Log.d(TAG, "로고 설정: ${responsiveSizes.logoSize}x${responsiveSizes.logoSize}dp")
        }

        // 3. 간격 요소 크기 조정
        adjustSpaces(view)

        // 4. 텍스트 크기 조정
        adjustTextSizes(view)

        Log.d(TAG, "반응형 크기 적용 완료: ${responsiveSizes}")
    }

    // 공간(여백) 뷰의 크기를 반응형으로 조정
    private fun adjustSpaces(view: View) {
        view.findViewById<View>(R.id.topSpace)?.let { space ->
            val params = space.layoutParams
            params.height = dpToPx(responsiveSizes.gap12dp.toFloat() / 2).toInt() // 6dp로 설정
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

    // 텍스트뷰에 반응형 글자 크기를 적용
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

    // dp 값을 px 값으로 변환
    private fun dpToPx(dp: Float): Float {
        return TypedValue.applyDimension(
            TypedValue.COMPLEX_UNIT_DIP,
            dp,
            context.resources.displayMetrics
        )
    }

    // 배경을 동적으로 생성 및 적용
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

    // 오버레이 뷰를 화면에서 제거
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

    // 오버레이의 레이아웃 속성 설정
    private fun createLayoutParams(): WindowManager.LayoutParams {
        val params = WindowManager.LayoutParams(
            WindowManager.LayoutParams.WRAP_CONTENT,
            WindowManager.LayoutParams.WRAP_CONTENT,
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY // Android 8.0 이상
            } else {
                @Suppress("DEPRECATION")
                WindowManager.LayoutParams.TYPE_PHONE // Android 8.0 미만
            },
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or // 포커스를 받지 않음
                    WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL or // 외부 터치 이벤트도 받음
                    WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH, // 외부 터치 감지
            PixelFormat.TRANSLUCENT // 배경 투명 처리
        )
        params.gravity = Gravity.TOP or Gravity.START // 왼쪽 위 모서리 기준 위치
        params.x = 100 // 초기 X 위치
        params.y = 100 // 초기 Y 위치
        return params
    }

    // 현재 오버레이 뷰 반환
    fun getOverlayView(): View? = overlayView

    // 현재 오버레이 레이아웃 파라미터 반환
    fun getLayoutParams(): WindowManager.LayoutParams? = layoutParams

    // 오버레이가 이미 생성되어 있는지 여부 확인
    fun isOverlayCreated(): Boolean = overlayView != null
}

package com.example.dynamic.service // 패키지 선언: 서비스 관련 클래스가 위치한 패키지

import android.app.Service // 안드로이드 서비스 기능 사용을 위한 import
import android.content.Context
import android.content.Intent
import android.os.Binder
import android.os.IBinder
import android.util.Log
import android.view.View
import android.view.WindowManager

// 백그라운드에서 실행되는 오버레이 서비스를 정의
class OverlayService : Service() {

    private var overlayManager: OverlayManager? = null // 오버레이 뷰 생성 및 제어 클래스
    private var dragHandler: DragHandler? = null // 오버레이 뷰 드래그 처리 클래스
    private var isOverlayVisible = false // 현재 오버레이가 화면에 보이는지 여부를 추적

    // 다른 컴포넌트에서 서비스에 접근할 수 있도록 바인더 제공
    private val binder = LocalBinder()

    // 내부 클래스: 바인더를 통해 서비스 인스턴스 접근 가능
    inner class LocalBinder : Binder() {
        fun getService(): OverlayService = this@OverlayService // 현재 서비스 반환
    }

    companion object {
        private const val TAG = "OverlayService" // 로그 태그

        // SharedPreferences 키 정의
        private const val PREF_NAME = "overlay_prefs" // SharedPreferences 파일 이름
        private const val KEY_OVERLAY_FLAG = "overlay_flag" // 오버레이 표시 여부를 저장하는 키
    }

    // ViewModel이나 Activity에서 서비스에 바인딩할 때 호출됨
    override fun onBind(intent: Intent?): IBinder = binder

    // 서비스가 생성될 때 호출됨
    override fun onCreate() {
        super.onCreate()
        Log.d(TAG, "OverlayService onCreate") // 로그 출력
        setupOverlay() // 오버레이 구성 및 초기화
    }

    // startService로 시작되었을 때 실행됨
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        return START_STICKY // 시스템이 종료하더라도 자동으로 다시 시작되도록 지정
    }

    // 서비스 종료 시 호출됨
    override fun onDestroy() {
        super.onDestroy()
        Log.d(TAG, "OverlayService onDestroy")
        cleanupOverlay() // 리소스 정리
    }

    // SharedPreferences에서 오버레이 표시 플래그 값을 가져옴
    private fun getOverlayFlag(): Int {
        val prefs = getSharedPreferences(PREF_NAME, Context.MODE_PRIVATE)
        return prefs.getInt(KEY_OVERLAY_FLAG, 0) // 기본값은 0 (숨김)
    }

    // 오버레이 표시 플래그 값을 저장
    fun setOverlayFlag(flag: Int) {
        val prefs = getSharedPreferences(PREF_NAME, Context.MODE_PRIVATE)
        prefs.edit().putInt(KEY_OVERLAY_FLAG, flag).apply() // 값 저장 및 반영

        Log.d(TAG, "플래그 변경됨: $flag")

        // 플래그 변경에 따라 오버레이 새로 생성 또는 제거
        if (flag == 1) {
            recreateOverlay() // 새로 생성
        } else {
            cleanupOverlay() // 제거
        }
    }

    // 현재 뷰의 실제 visibility 상태 확인
    private fun getActualViewVisibility(): String {
        val view = overlayManager?.getOverlayView()
        return when (view?.visibility) {
            View.VISIBLE -> "VISIBLE"
            View.INVISIBLE -> "INVISIBLE"
            View.GONE -> "GONE"
            else -> "NULL" // 뷰가 null일 경우
        }
    }

    // 플래그 값만으로 오버레이 표시 여부를 결정 (중앙 제어 함수)
    private fun updateOverlayVisibility() {
        val currentFlag = getOverlayFlag() // 현재 저장된 플래그 값
        val shouldShow = currentFlag == 1 // 1이면 보여야 함
        val actualVisibility = getActualViewVisibility() // 뷰 상태 확인

        Log.d(TAG, "상태 체크 - 플래그: $currentFlag, 추적된오버레이: $isOverlayVisible")
        Log.d(TAG, "실제 View visibility: $actualVisibility")
        Log.d(TAG, "표시해야함: $shouldShow (플래그만으로 결정)")

        val actuallyVisible = (actualVisibility == "VISIBLE")
        if (actuallyVisible != isOverlayVisible) {
            Log.e(TAG, "상태 불일치! 추적: $isOverlayVisible, 실제: $actuallyVisible")
            isOverlayVisible = actuallyVisible // 실제 상태로 동기화
        }

        if (shouldShow && !isOverlayVisible) {
            showOverlay() // 표시 필요 → 표시
        } else if (!shouldShow && isOverlayVisible) {
            hideOverlay() // 숨김 필요 → 숨김
        } else {
            Log.d(TAG, "상태 변화 없음") // 그대로 유지
        }

        Log.d(TAG, "최종 상태 - 추적: $isOverlayVisible, 실제: ${getActualViewVisibility()}")
    }

    // 오버레이 구성 초기화 (최초 1회만 실행)
    private fun setupOverlay() {
        val windowManager = getSystemService(WINDOW_SERVICE) as WindowManager
        overlayManager = OverlayManager(this, windowManager) // 오버레이 관리 클래스 초기화
        dragHandler = DragHandler(windowManager, resources, this) // 드래그 이벤트 처리 클래스 초기화
        createOverlay() // 오버레이 생성 (처음은 숨김 상태)
    }

    // 오버레이 재생성 (플래그 변경 시)
    private fun recreateOverlay() {
        Log.d(TAG, "오버레이 재생성 시작")
        cleanupOverlay() // 기존 제거
        setupOverlay() // 새로 생성
        updateOverlayVisibility() // 플래그에 따라 표시 여부 결정
        Log.d(TAG, "오버레이 재생성 완료")
    }

    // 오버레이 생성 및 화면에 추가
    private fun createOverlay() {
        val touchListener = createTouchListener() // 드래그 가능 리스너 생성
        val result = overlayManager?.createOverlay(touchListener) // 오버레이 생성

        if (result == null) {
            Log.e(TAG, "오버레이 생성 실패")
            stopSelf() // 서비스 종료
        } else {
            val (view, params) = result // 생성된 뷰와 레이아웃 파라미터 분리
            Log.d(TAG, "오버레이 생성 성공 (XML inflate)")
            Log.d(TAG, "생성 직후 View visibility: ${getActualViewVisibility()}")
            forceHideOverlay() // 기본적으로 숨김 상태로 시작
        }
    }

    // 오버레이를 움직일 수 있게 만드는 터치 리스너 생성
    private fun createTouchListener(): View.OnTouchListener {
        return View.OnTouchListener { _, event ->
            val overlayView = overlayManager?.getOverlayView()
            val layoutParams = overlayManager?.getLayoutParams()

            if (overlayView != null && layoutParams != null) {
                dragHandler?.handleTouchEvent(event, overlayView, layoutParams) ?: false
            } else {
                false
            }
        }
    }

    // 오버레이 화면에 표시
    private fun showOverlay() {
        val view = overlayManager?.getOverlayView()
        if (view != null && !isOverlayVisible) {
            view.visibility = View.VISIBLE
            isOverlayVisible = true
            Log.d(TAG, "오버레이 표시됨 (isOverlayVisible = true)")
            Log.d(TAG, "표시 후 실제 visibility: ${getActualViewVisibility()}")
        }
    }

    // 오버레이 화면에서 숨김
    private fun hideOverlay() {
        val view = overlayManager?.getOverlayView()
        if (view != null && isOverlayVisible) {
            view.visibility = View.GONE
            isOverlayVisible = false
            Log.d(TAG, "오버레이 숨겨짐 (isOverlayVisible = false)")
            Log.d(TAG, "숨김 후 실제 visibility: ${getActualViewVisibility()}")
        }
    }

    // 강제로 오버레이 숨기기 (초기 상태용)
    private fun forceHideOverlay() {
        val view = overlayManager?.getOverlayView()
        if (view != null) {
            view.visibility = View.GONE
            isOverlayVisible = false
            Log.d(TAG, "강제 숨김 완료 (isOverlayVisible = false)")
            Log.d(TAG, "강제 숨김 후 실제 visibility: ${getActualViewVisibility()}")
        }
    }

    // 오버레이 리소스 및 상태 정리
    private fun cleanupOverlay() {
        overlayManager?.removeOverlay() // 오버레이 뷰 제거
        overlayManager = null
        dragHandler = null
        isOverlayVisible = false
        Log.d(TAG, "오버레이 정리 완료")
    }

    // 현재 오버레이가 화면에 존재하는지 여부 확인 (생성 여부 기준)
    fun isOverlayActive(): Boolean {
        return overlayManager?.isOverlayCreated() == true && getOverlayFlag() == 1
    }

    // 오버레이가 현재 표시 중인지 여부 확인 (플래그와 상태 함께 확인)
    fun isOverlayVisible(): Boolean = isOverlayVisible && getOverlayFlag() == 1
}

## dynamic.service.OverlayService
package com.example.dynamic.service

import android.app.Service
import android.content.Context
import android.content.Intent
import android.os.Binder
import android.os.IBinder
import android.util.Log
import android.view.View
import android.view.WindowManager

class OverlayService : Service() {

    // OverlayManager 인스턴스를 저장할 변수. 오버레이 생성/제거, 뷰 접근 등을 관리함
    private var overlayManager: OverlayManager? = null

    // DragHandler 인스턴스를 저장할 변수. 오버레이를 사용자가 드래그할 수 있게 해줌
    private var dragHandler: DragHandler? = null

    // 오버레이가 현재 보이는 상태인지 추적하기 위한 변수 (내부 추적용)
    private var isOverlayVisible = false

    // 외부에서 서비스에 바인딩할 수 있게 하는 LocalBinder 객체 (뷰모델 등과 연동 가능)
    private val binder = LocalBinder()

    // 내부 바인더 클래스. 외부에서 서비스 인스턴스를 얻기 위해 사용됨
    inner class LocalBinder : Binder() {
        fun getService(): OverlayService = this@OverlayService // 바깥의 서비스 인스턴스를 반환
    }

    companion object {
        // 로그 출력 시 태그로 사용됨
        private const val TAG = "OverlayService"

        // SharedPreferences 파일명과 키값 정의
        private const val PREF_NAME = "overlay_prefs"
        private const val KEY_OVERLAY_FLAG = "overlay_flag"
    }

    // 클라이언트가 서비스에 바인딩할 때 호출됨. 바인더 객체를 반환
    override fun onBind(intent: Intent?): IBinder = binder

    // 서비스가 처음 생성될 때 호출됨. 오버레이 초기화 수행
    override fun onCreate() {
        super.onCreate()
        Log.d(TAG, "OverlayService onCreate") // 로그 기록
        setupOverlay() // 오버레이 초기 설정 및 준비
    }

    // 서비스가 명령(Intent)에 의해 시작될 때 호출됨
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // START_STICKY: 시스템이 서비스를 죽였을 때 자동 재시작 요청
        return START_STICKY
    }

    // 서비스가 종료될 때 호출됨. 오버레이 제거 및 리소스 정리
    override fun onDestroy() {
        super.onDestroy()
        Log.d(TAG, "OverlayService onDestroy") // 로그 기록
        cleanupOverlay() // 생성한 오버레이 뷰 제거 및 메모리 정리
    }
}

// 현재 플래그 상태 확인
private fun getOverlayFlag(): Int {
    // "overlay_prefs"라는 이름의 SharedPreferences를 앱 내부 전용 모드(Context.MODE_PRIVATE)로 연다.
    val prefs = getSharedPreferences(PREF_NAME, Context.MODE_PRIVATE)
    // 저장된 오버레이 플래그 값을 가져오며, 값이 없을 경우 기본값 0을 반환한다.
    return prefs.getInt(KEY_OVERLAY_FLAG, 0)
}

// 플래그 상태 저장
fun setOverlayFlag(flag: Int) {
    // "overlay_prefs"라는 SharedPreferences 객체를 열어 현재 오버레이 상태를 저장할 준비를 한다.
    val prefs = getSharedPreferences(PREF_NAME, Context.MODE_PRIVATE)
    // 오버레이 상태 값을 KEY_OVERLAY_FLAG에 저장하고, apply()를 통해 비동기로 커밋한다.
    prefs.edit().putInt(KEY_OVERLAY_FLAG, flag).apply()
    
    // 현재 설정한 플래그 값을 로그로 출력하여 디버깅 용도로 활용한다.
    Log.d(TAG, "플래그 변경됨: $flag")
    
    // 플래그가 1이면 오버레이를 새롭게 생성(recreate), 0이면 기존 오버레이를 제거(cleanup)한다.
    // 즉, 플래그 값에 따라 오버레이가 화면에 보일지 여부를 결정하는 핵심 분기이다.
    if (flag == 1) {
        recreateOverlay()
    } else {
        cleanupOverlay()
    }
}


//실제 View의 visibility 상태 확인
private fun getActualViewVisibility(): String {
    // overlayManager로부터 현재 오버레이 View 객체를 가져온다. 없을 경우 null이다.
    val view = overlayManager?.getOverlayView()
    
    // View 객체의 visibility 속성 값에 따라 상태 문자열을 반환한다.
    return when (view?.visibility) {
        View.VISIBLE -> "VISIBLE"    // View가 사용자에게 보이는 상태
        View.INVISIBLE -> "INVISIBLE" // View가 보이지 않지만 layout은 유지되는 상태
        View.GONE -> "GONE"          // View가 보이지 않고 layout에서도 제외된 상태
        else -> "NULL"               // View 객체 자체가 없을 경우
    }
}

/**
 * 오버레이 visibility 업데이트 (핵심 로직)
 * 플래그 상태만으로 오버레이 표시/숨김 제어 (앱 상태 무관)
 */
private fun updateOverlayVisibility() {
    // SharedPreferences에서 현재 오버레이 표시 플래그 값을 가져온다.
    val currentFlag = getOverlayFlag()
    
    // 플래그가 1이면 오버레이를 보여야 하므로 true, 0이면 false
    val shouldShow = currentFlag == 1  // 앱 상태나 사용자 입력과 무관하게 오직 플래그로만 제어
    
    // 실제 View의 visibility 값을 문자열로 가져온다 (VISIBLE, GONE 등)
    val actualVisibility = getActualViewVisibility()
    
    // 현재 상태 로그 출력: 플래그 값과 내부 추적 변수 값을 함께 출력
    Log.d(TAG, "상태 체크 - 플래그: $currentFlag, 추적된오버레이: $isOverlayVisible")
    Log.d(TAG, "실제 View visibility: $actualVisibility")
    Log.d(TAG, "표시해야함: $shouldShow (플래그만으로 결정)")
    
    // 실제 View가 보이는 상태인지 Boolean으로 계산
    val actuallyVisible = (actualVisibility == "VISIBLE")
    
    // 내부에서 추적 중인 상태(isOverlayVisible)와 실제 View 상태가 다르면 경고 로그 출력
    if (actuallyVisible != isOverlayVisible) {
        Log.e(TAG, "상태 불일치! 추적: $isOverlayVisible, 실제: $actuallyVisible")
        
        // 내부 상태를 실제 View 상태로 동기화하여 상태 일관성 유지
        isOverlayVisible = actuallyVisible
    }

    // 플래그에 따라 오버레이를 표시해야 하지만, 현재 안 보이는 경우 → 보여주기
    if (shouldShow && !isOverlayVisible) {
        Log.d(TAG, "오버레이 표시 (플래그=1)")
        showOverlay()
    
    // 반대로, 플래그는 숨김이지만 실제 View는 보이는 경우 → 숨기기
    } else if (!shouldShow && isOverlayVisible) {
        Log.d(TAG, "오버레이 숨김 (플래그=0)")
        hideOverlay()

    // 상태가 일치하는 경우 → 아무 동작도 하지 않음
    } else {
        Log.d(TAG, "상태 변화 없음")
    }
    
    // 최종적으로 상태가 일치하는지 확인용 로그 출력
    Log.d(TAG, "최종 상태 - 추적: $isOverlayVisible, 실제: ${getActualViewVisibility()}")
}




/**
 * 오버레이 설정 및 생성
 */
private fun setupOverlay() {
    // 시스템 서비스에서 WindowManager 인스턴스를 가져온다. 이는 오버레이를 화면에 띄우기 위해 필요하다.
    val windowManager = getSystemService(WINDOW_SERVICE) as WindowManager
    
    // OverlayManager 인스턴스를 생성하면서 context와 windowManager를 전달한다.
    // 이 클래스는 오버레이 View의 생성과 제거를 책임진다.
    overlayManager = OverlayManager(this, windowManager)
    
    // DragHandler 인스턴스를 생성하여 드래그 기능을 위한 터치 이벤트 핸들링을 가능하게 한다.
    dragHandler = DragHandler(windowManager, resources, this)
    
    // 오버레이 View를 생성하되, 처음에는 보이지 않는 상태(GONE)로 생성한다.
    createOverlay()
}

/**
 * 오버레이 완전히 재생성 (새로운 XML 적용)
 */
private fun recreateOverlay() {
    // 디버깅을 위해 로그 출력: 오버레이를 다시 만들기 시작함
    Log.d(TAG, "오버레이 재생성 시작")
    
    // 기존에 생성된 오버레이를 완전히 제거하여 중복 생성 또는 충돌을 방지
    cleanupOverlay()
    
    // 오버레이를 새롭게 설정하고 생성한다 (OverlayManager와 DragHandler도 다시 초기화됨)
    setupOverlay()
    
    // 현재 저장된 플래그 값에 따라 오버레이를 보여줄지 숨길지 결정한다
    updateOverlayVisibility()
    
    // 디버깅을 위한 로그 출력: 재생성이 끝났음을 알림
    Log.d(TAG, "오버레이 재생성 완료")
}

/**
 * 오버레이 생성
 */
private fun createOverlay() {
    // DragHandler에서 사용할 터치 리스너를 생성한다 (사용자 터치 이벤트 대응)
    val touchListener = createTouchListener()
    
    // overlayManager를 통해 오버레이를 생성하고 View와 LayoutParams를 받아온다
    val result = overlayManager?.createOverlay(touchListener)
    
    // 만약 result가 null이라면 생성에 실패한 것이므로 로그 출력 후 서비스 종료
    if (result == null) {
        Log.e(TAG, "오버레이 생성 실패")
        stopSelf() // 서비스 강제 종료
    } else {
        // 정상적으로 생성되었을 경우 View와 레이아웃 파라미터를 구조 분해 할당
        val (view, params) = result
        
        // 생성 성공 로그 출력 및 View의 visibility 상태를 출력
        Log.d(TAG, "오버레이 생성 성공 (XML inflate)")
        Log.d(TAG, "생성 직후 View visibility: ${getActualViewVisibility()}")
        
        // 서비스 시작 시 오버레이는 기본적으로 숨겨진 상태여야 하므로 강제 숨김 처리
        forceHideOverlay()
    }
}


@SuppressLint("ClickableViewAccessibility")
// 접근성 경고 무시 - 사용자 정의 터치 리스너 사용 시 TalkBack 등 경고 발생 방지
private fun createTouchListener(): View.OnTouchListener {
    return View.OnTouchListener { _, event ->
        val overlayView = overlayManager?.getOverlayView() // 현재 오버레이 뷰 가져오기
        val layoutParams = overlayManager?.getLayoutParams() // 해당 뷰의 레이아웃 파라미터 가져오기

        if (overlayView != null && layoutParams != null) {
            // 두 값이 null이 아니면 DragHandler로 터치 이벤트 위임
            dragHandler?.handleTouchEvent(event, overlayView, layoutParams) ?: false
        } else {
            false // 뷰나 파라미터가 null이면 이벤트 처리 안 함
        }
    }
}

// 오버레이 표시 함수
private fun showOverlay() {
    val view = overlayManager?.getOverlayView() // 오버레이 뷰 가져오기
    if (view != null && !isOverlayVisible) { // 뷰가 존재하고 아직 표시되지 않았다면
        view.visibility = View.VISIBLE // 화면에 표시
        isOverlayVisible = true // 내부 상태 플래그도 표시로 변경
        Log.d(TAG, "오버레이 표시됨 (isOverlayVisible = true)")
        Log.d(TAG, "표시 후 실제 visibility: ${getActualViewVisibility()}")
    }
}

// 오버레이 숨김 함수
private fun hideOverlay() {
    val view = overlayManager?.getOverlayView() // 오버레이 뷰 가져오기
    if (view != null && isOverlayVisible) { // 뷰가 존재하고 현재 표시 상태라면
        view.visibility = View.GONE // 화면에서 제거 (숨김)
        isOverlayVisible = false // 내부 상태도 false로 변경
        Log.d(TAG, "오버레이 숨겨짐 (isOverlayVisible = false)")
        Log.d(TAG, "숨김 후 실제 visibility: ${getActualViewVisibility()}")
    }
}

// 오버레이를 강제로 숨기는 함수 (앱 시작 시 초기화용)
private fun forceHideOverlay() {
    val view = overlayManager?.getOverlayView() // 오버레이 뷰 가져오기
    if (view != null) {
        view.visibility = View.GONE // 무조건 숨김 처리
        isOverlayVisible = false // 내부 상태도 false로 초기화
        Log.d(TAG, "강제 숨김 완료 (isOverlayVisible = false)")
        Log.d(TAG, "강제 숨김 후 실제 visibility: ${getActualViewVisibility()}")
    }
}

// 오버레이 뷰 및 핸들러 제거, 상태 초기화
private fun cleanupOverlay() {
    overlayManager?.removeOverlay() // WindowManager에서 오버레이 제거
    overlayManager = null // manager 객체 null 처리
    dragHandler = null // drag handler도 제거
    isOverlayVisible = false // 표시 상태 false로 초기화
    Log.d(TAG, "오버레이 정리 완료")
}

// 오버레이가 생성되었고 플래그가 활성화 상태인지 확인
fun isOverlayActive(): Boolean {
    return overlayManager?.isOverlayCreated() == true && getOverlayFlag() == 1
    // 실제 뷰가 생성되었고, SharedPreferences 상에서도 플래그가 1이면 true
}

// 현재 오버레이가 화면에 표시 중인지 확인하는 함수
fun isOverlayVisible(): Boolean = isOverlayVisible && getOverlayFlag() == 1
// 내부 상태 플래그와 SharedPreferences 플래그가 모두 true여야 표시 중으로 판단

## dynamic.viewmodel.OverlayViewModel
package com.example.dynamic.viewmodel

import android.app.Application
import android.content.ComponentName
import android.content.Context
import android.content.Intent
import android.content.ServiceConnection
import android.os.IBinder
import androidx.lifecycle.AndroidViewModel
import androidx.lifecycle.LiveData
import androidx.lifecycle.MutableLiveData
import com.example.dynamic.DynamicApplication
import com.example.dynamic.service.OverlayService

// AndroidViewModel을 상속받은 뷰모델 클래스, Application Context에 접근 가능
class OverlayViewModel(application: Application) : AndroidViewModel(application) {

    private val context = application.applicationContext // 앱 전체에서 사용 가능한 Context 획득
    private val app = application as DynamicApplication // 커스텀 Application 클래스 참조 (start/stop 서비스용)

    // SharedPreferences 객체: 오버레이 상태를 영속적으로 저장/불러오기 위해 사용
    private val prefs = context.getSharedPreferences("overlay_prefs", Context.MODE_PRIVATE)

    // 오버레이 플래그 값을 감싸는 LiveData (UI에서 옵저빙 가능)
    private val _overlayFlag = MutableLiveData<Int>()
    val overlayFlag: LiveData<Int> = _overlayFlag // 외부에 노출할 ReadOnly LiveData

    // 바인딩된 OverlayService를 저장할 변수
    private var overlayService: OverlayService? = null
    private var isServiceBound = false // 서비스가 현재 바인딩되어 있는지 여부

    // 서비스 연결 콜백 객체
    private val serviceConnection = object : ServiceConnection {
        // 서비스와 성공적으로 연결되었을 때 호출
        override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
            val binder = service as? OverlayService.LocalBinder // LocalBinder 캐스팅
            overlayService = binder?.getService() // 실제 OverlayService 객체 획득
            isServiceBound = true

            // 연결 후 현재 뷰모델에 저장된 플래그를 서비스에도 반영
            val currentFlag = _overlayFlag.value ?: 0
            overlayService?.setOverlayFlag(currentFlag)
        }

        // 예기치 않게 연결이 끊겼을 때 호출
        override fun onServiceDisconnected(name: ComponentName?) {
            overlayService = null // 서비스 참조 제거
            isServiceBound = false // 연결 상태 false로 변경
        }
    }

    init {
        // 뷰모델 초기화 시 SharedPreferences에서 마지막 저장된 플래그 불러옴
        val savedFlag = prefs.getInt("overlay_flag", 0)
        _overlayFlag.value = savedFlag // LiveData에 저장해 UI에 즉시 반영

        // 저장된 플래그가 1이면 앱이 꺼졌어도 오버레이 다시 활성화
        if (savedFlag == 1) {
            startOverlayService()
        }
    }

    /**
     * 오버레이 플래그 값을 토글 (0이면 1로, 1이면 0으로)
     */
    fun toggleOverlayFlag() {
        val currentFlag = _overlayFlag.value ?: 0
        val newFlag = if (currentFlag == 0) 1 else 0
        setOverlayFlag(newFlag)
    }

    /**
     * 오버레이 플래그 설정
     */
    private fun setOverlayFlag(flag: Int) {
        _overlayFlag.value = flag // UI 반영을 위해 LiveData 갱신
        prefs.edit().putInt("overlay_flag", flag).apply() // SharedPreferences에 저장

        if (flag == 1) {
            startOverlayService() // 1이면 서비스 시작
        } else {
            stopOverlayService() // 0이면 서비스 중단
        }
    }

    /**
     * 오버레이 서비스 시작 및 바인딩
     */
    private fun startOverlayService() {
        app.startOverlayService() // Application 객체를 통해 서비스 시작

        if (!isServiceBound) {
            val intent = Intent(context, OverlayService::class.java) // 바인딩할 Intent 생성
            context.bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE)
            // Context.BIND_AUTO_CREATE: 없으면 자동으로 생성하여 연결
        }
    }

    /**
     * 오버레이 서비스 중지 및 바인딩 해제
     */
    private fun stopOverlayService() {
        if (isServiceBound) {
            context.unbindService(serviceConnection) // 바인딩 해제
            isServiceBound = false
            overlayService = null
        }

        app.stopOverlayService() // Application 객체를 통해 서비스 완전히 중지
    }

    /**
     * 현재 오버레이가 활성화된 상태인지 확인 (플래그가 1인지 여부)
     */
    fun isOverlayActive(): Boolean {
        return (_overlayFlag.value ?: 0) == 1
    }

    /**
     * 오버레이 상태를 설명하는 텍스트 반환 (UI 출력용)
     */
    fun getOverlayStatusText(): String {
        val flag = _overlayFlag.value ?: 0
        return if (flag == 1) {
            "오버레이가 활성화 (플래그: $flag)"
        } else {
            "오버레이가 비활성화 (플래그: $flag)"
        }
    }

    /**
     * 뷰모델이 파괴될 때 호출되는 콜백 - 서비스 정리
     */
    override fun onCleared() {
        super.onCleared()
        if (isServiceBound) {
            context.unbindService(serviceConnection) // 메모리 누수 방지 위해 서비스 해제
            isServiceBound = false
        }
    }
}

## dynamic.DynamicApplication
package com.example.dynamic
// 이 파일이 속한 패키지를 정의합니다.

import android.app.Application
// Application 클래스를 사용하기 위해 import합니다. 앱 전체의 라이프사이클을 담당합니다.

import android.content.Intent
// 서비스 시작에 필요한 Intent 클래스를 사용하기 위해 import합니다.

import android.util.Log
// 로그 출력용 Log 클래스를 사용하기 위해 import합니다.

import com.example.dynamic.service.OverlayService
// 우리가 사용할 OverlayService 클래스를 import합니다.

class DynamicApplication : Application() {
// Application 클래스를 상속받아 앱 전체에서 사용할 전역 설정을 관리하는 클래스입니다.

    private var isOverlayServiceRunning = false
    // 오버레이 서비스가 실행 중인지 여부를 추적하는 Boolean 변수입니다.

    companion object {
        private const val TAG = "DynamicApplication"
        // 로그 출력 시 태그로 사용할 상수를 정의합니다.
    }

    override fun onCreate() {
        super.onCreate()
        // Application이 처음 생성될 때 호출되는 메서드로 초기 설정이 들어갑니다.

        Log.d(TAG, "DynamicApplication onCreate")
        // 로그를 통해 Application이 시작되었음을 알립니다.
    }

    /**
     * 오버레이 서비스 시작 (외부에서 호출)
     */
    fun startOverlayService() {
        if (!isOverlayServiceRunning) {
            // 서비스가 아직 실행 중이 아니라면 실행합니다.
            Log.d(TAG, "오버레이 서비스 시작")
            val intent = Intent(this, OverlayService::class.java)
            // OverlayService를 실행하기 위한 인텐트를 생성합니다.
            startService(intent)
            // 서비스 시작
            isOverlayServiceRunning = true
            // 실행 상태 플래그 업데이트
        } else {
            Log.d(TAG, "오버레이 서비스 이미 실행 중")
            // 이미 실행 중이면 로그만 출력하고 중복 실행 방지
        }
    }

    /**
     * 오버레이 서비스 중지 (외부에서 호출)
     */
    fun stopOverlayService() {
        if (isOverlayServiceRunning) {
            // 실행 중일 경우에만 중지 시도
            Log.d(TAG, "오버레이 서비스 중지")
            val intent = Intent(this, OverlayService::class.java)
            // 종료할 서비스에 대한 인텐트를 생성
            stopService(intent)
            // 서비스 중지
            isOverlayServiceRunning = false
            // 플래그 false로 설정
        } else {
            Log.d(TAG, "오버레이 서비스 이미 중지됨")
            // 이미 중지 상태라면 로그만 출력
        }
    }

    /**
     * 오버레이 서비스 실행 상태 확인
     */
    fun isOverlayServiceRunning(): Boolean = isOverlayServiceRunning
    // 현재 오버레이 서비스가 실행 중인지 Boolean 값으로 반환합니다.
}

## dynamic.DynamicApplication2
package com.example.dynamic // 패키지 선언: 프로젝트 내 위치를 지정

import android.app.Application // Android의 Application 클래스 import
import android.content.Intent // 서비스 실행에 사용할 Intent 클래스 import
import android.util.Log // 로그 출력을 위한 Log 클래스 import
import com.example.dynamic.service.OverlayService // 우리가 정의한 오버레이 서비스 import

// 앱 전체에서 공통적으로 접근할 수 있는 전역 설정 및 상태를 관리하는 Application 클래스 상속
class DynamicApplication : Application() {

    private var isOverlayServiceRunning = false // 오버레이 서비스가 현재 실행 중인지 여부를 나타내는 변수

    companion object {
        private const val TAG = "DynamicApplication" // 로그 출력에 사용할 태그 상수
    }

    // 애플리케이션이 처음 생성될 때 호출되는 메서드 (앱이 시작될 때 1회 호출됨)
    override fun onCreate() {
        super.onCreate()
        Log.d(TAG, "DynamicApplication onCreate") // 애플리케이션 시작 시 로그 출력
    }

    // 오버레이 서비스를 시작하는 메서드 (다른 클래스에서 호출 가능)
    fun startOverlayService() {
        // 서비스가 실행 중이 아닐 경우에만 실행
        if (!isOverlayServiceRunning) {
            Log.d(TAG, "오버레이 서비스 시작") // 시작 로그 출력

            // OverlayService를 실행하기 위한 명시적 인텐트 생성
            val intent = Intent(this, OverlayService::class.java)

            // 서비스 실행 요청 (foreground X, background 서비스로 실행)
            startService(intent)

            // 서비스 실행 상태를 true로 업데이트
            isOverlayServiceRunning = true
        } else {
            // 이미 실행 중인 경우 로그 출력
            Log.d(TAG, "오버레이 서비스 이미 실행 중")
        }
    }

    // 오버레이 서비스를 중지하는 메서드 (다른 클래스에서 호출 가능)
    fun stopOverlayService() {
        // 서비스가 실행 중일 때만 중지 진행
        if (isOverlayServiceRunning) {
            Log.d(TAG, "오버레이 서비스 중지") // 중지 로그 출력

            // 중지할 서비스를 지정하기 위한 명시적 인텐트 생성
            val intent = Intent(this, OverlayService::class.java)

            // 서비스 중지 요청
            stopService(intent)

            // 서비스 실행 상태를 false로 업데이트
            isOverlayServiceRunning = false
        } else {
            // 이미 중지된 상태일 경우 로그 출력
            Log.d(TAG, "오버레이 서비스 이미 중지됨")
        }
    }

    // 오버레이 서비스가 현재 실행 중인지 확인하는 메서드 (true 또는 false 반환)
    fun isOverlayServiceRunning(): Boolean = isOverlayServiceRunning
}

## dynamic.MainApplication
package com.example.dynamic // 패키지 이름 선언

import android.content.Intent // 인텐트 관련 클래스 import
import android.net.Uri // URI(패키지 경로) 관련 클래스 import
import android.os.Build // SDK 버전 확인용
import android.os.Bundle // 액티비티 생명주기 전달 데이터
import android.provider.Settings // 오버레이 설정으로 이동할 때 사용
import android.widget.Button // 버튼 View import
import android.widget.TextView // 텍스트뷰 View import
import android.widget.Toast // 토스트 메시지 출력용
import androidx.activity.result.contract.ActivityResultContracts // 권한 설정 결과 수신을 위한 Contract
import androidx.activity.viewModels // ViewModel 연결
import androidx.appcompat.app.AppCompatActivity // 기본 AppCompatActivity
import androidx.lifecycle.Observer // LiveData 관찰용
import com.example.dynamic.viewmodel.OverlayViewModel // 우리가 만든 오버레이 ViewModel import

// 앱의 메인 화면 Activity
class MainActivity : AppCompatActivity() {

    private val viewModel: OverlayViewModel by viewModels() // ViewModel을 Activity에 바인딩 (생명주기 연동)

    // View 참조 변수 선언 (나중에 초기화)
    private lateinit var statusText: TextView
    private lateinit var permissionStatusText: TextView
    private lateinit var toggleButton: Button

    private var hasShownPermissionToast = false // 토스트 중복 방지를 위한 플래그

    // 오버레이 권한 설정 후 결과를 처리하는 런처 등록
    private val permissionLauncher = registerForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { _ ->
        // 권한 설정 창에서 돌아왔을 때 UI 상태 재확인
        updateUIBasedOnPermission()
    }

    // 액티비티 생성 시 호출되는 생명주기 함수
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        supportActionBar?.hide() // 상단 액션바 숨기기 (타이틀 중복 방지)

        setContentView(R.layout.activity_main) // 레이아웃 XML 설정

        initViews() // 뷰 참조 초기화
        setupViews() // 이벤트 리스너 설정
        observeViewModel() // ViewModel LiveData 관찰 시작

        updateUIBasedOnPermission() // 시작 시 권한 및 UI 상태 반영
    }

    // findViewById를 통해 뷰를 참조해 변수에 저장
    private fun initViews() {
        statusText = findViewById(R.id.statusText)
        permissionStatusText = findViewById(R.id.permissionStatusText)
        toggleButton = findViewById(R.id.toggleButton)
    }

    // 버튼에 클릭 리스너 부착
    private fun setupViews() {
        toggleButton.setOnClickListener {
            handleToggleButtonClick() // 버튼 클릭 시 실행할 동작 정의
        }
    }

    // 토글 버튼 클릭 시 처리 로직
    private fun handleToggleButtonClick() {
        if (hasOverlayPermission()) {
            viewModel.toggleOverlayFlag() // 권한이 있으면 플래그 토글
        } else {
            requestOverlayPermission() // 없으면 권한 요청
        }
    }

    // ViewModel의 LiveData를 관찰하여 UI 업데이트
    private fun observeViewModel() {
        viewModel.overlayFlag.observe(this, Observer { flag ->
            if (hasOverlayPermission()) {
                updateUIWithFlag(flag) // 권한이 있을 때만 UI 업데이트
            }
        })
    }

    // 권한 여부에 따라 전체 UI 상태를 업데이트
    private fun updateUIBasedOnPermission() {
        if (hasOverlayPermission()) {
            updatePermissionStatusUI(true) // 권한 있음 표시
            val currentFlag = viewModel.overlayFlag.value ?: 0
            updateUIWithFlag(currentFlag) // 현재 오버레이 플래그 반영
        } else {
            updatePermissionStatusUI(false) // 권한 없음 표시
            updateUIForPermissionRequired() // 권한 요청 UI로 변경
        }
    }

    // 텍스트뷰를 통해 권한 상태 표시 (초록/빨강)
    private fun updatePermissionStatusUI(hasPermission: Boolean) {
        if (hasPermission) {
            permissionStatusText.text = "권한 허용됨"
            permissionStatusText.setTextColor(getColor(android.R.color.holo_green_dark))
        } else {
            permissionStatusText.text = "권한 필요"
            permissionStatusText.setTextColor(getColor(android.R.color.holo_red_dark))
        }
    }

    // 플래그 값(0/1)에 따라 UI를 구성 (버튼 텍스트, 색상 등)
    private fun updateUIWithFlag(flag: Int) {
        val statusMessage = viewModel.getOverlayStatusText() // 상태 텍스트 생성
        val isActive = flag == 1

        statusText.text = statusMessage // 상태 텍스트뷰에 반영

        if (isActive) {
            toggleButton.text = "오버레이 끄기 (현재: $flag)"
            toggleButton.setBackgroundColor(getColor(android.R.color.holo_red_light))
        } else {
            toggleButton.text = "오버레이 켜기 (현재: $flag)"
            toggleButton.setBackgroundColor(getColor(android.R.color.holo_green_light))
        }

        toggleButton.isEnabled = true // 버튼 활성화
    }

    // 권한이 필요한 경우 UI 구성
    private fun updateUIForPermissionRequired() {
        statusText.text = "오버레이 권한이 필요합니다"
        toggleButton.text = "권한 요청"
        toggleButton.setBackgroundColor(getColor(android.R.color.holo_orange_light))
        toggleButton.isEnabled = true

        // 처음 한 번만 토스트 메시지 출력
        if (!hasShownPermissionToast) {
            Toast.makeText(this, "오버레이 권한을 허용해주세요", Toast.LENGTH_LONG).show()
            hasShownPermissionToast = true
        }
    }

    // 현재 오버레이 권한이 있는지 확인
    private fun hasOverlayPermission(): Boolean {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            Settings.canDrawOverlays(this) // 마시멜로 이상일 경우 시스템 API 사용
        } else {
            true // 하위 버전은 기본적으로 권한 허용됨
        }
    }

    // 시스템 설정으로 이동하여 사용자에게 권한 요청
    private fun requestOverlayPermission() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            val intent = Intent(
                Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
                Uri.parse("package:$packageName")
            )
            permissionLauncher.launch(intent) // 결과는 위의 permissionLauncher가 받음
        }
    }

    // 액티비티가 다시 화면에 나타날 때 (백 → 포그라운드)
    override fun onResume() {
        super.onResume()
        updateUIBasedOnPermission() // 포커싱될 때마다 권한 상태 재확인
    }
}






## manifest
    <!-- 다른 앱 위에 오버레이 뷰를 띄우기 위한 권한 선언 -->
    <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />



        <!-- 오버레이 서비스 등록 -->
        <service
            android:name=".service.OverlayService" <!-- 실행할 서비스 클래스 경로 -->
            android:enabled="true"                 <!-- 서비스 활성화 여부 -->
            android:exported="false" />            <!-- 외부 앱에서 이 서비스 실행 금지 (보안 목적) -->

</manifest>
이 AndroidManifest.xml은 어떤 코드와 연결되어 있나요?
android:name=".DynamicApplication"
→ DynamicApplication.kt 클래스와 연결됨. 앱 시작 시 이 클래스가 초기화됨.

<activity android:name=".MainActivity">
→ MainActivity.kt와 직접 연결됨. 사용자가 앱을 실행하면 이 액티비티가 시작됨.

<service android:name=".service.OverlayService">
→ OverlayService.kt와 연결됨. 오버레이 UI를 관리하는 서비스가 실행되도록 선언한 것.

<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
→ Settings.canDrawOverlays(context)를 통해 권한 확인 후 오버레이 실행이 가능하도록 선언한 것.

----
# 안드로이드 오버레이 앱 구성 및 실행 흐름 정리

## 1. DragHandler.kt
- 역할: 오버레이 뷰를 터치로 끌 수 있도록 이벤트를 처리
- 핵심 포인트:
  - `MotionEvent.ACTION_DOWN`, `MOVE`, `UP` 등을 통해 이동 좌표 계산
  - `rawX`, `rawY`는 화면 전체 기준 절대 좌표 (오버레이 이동에 필요)

## 2. OverlayManager.kt
- 역할: 오버레이 UI 생성, 제거 및 크기·레이아웃 파라미터 제어
- 핵심 포인트:
  - `WindowManager.LayoutParams` 생성
  - `GradientDrawable`로 원형 배경 생성 (임시 시각화용)
  - 뷰의 크기를 기준 화면 대비 동적으로 계산
  - 오버레이 뷰와 `LayoutParams` 반환

## 3. OverlayService.kt
- 역할: 오버레이 생명주기 관리, View 표시·숨김·정리 로직 처리
- 핵심 포인트:
  - `Service`에서만 `TYPE_APPLICATION_OVERLAY` 사용 가능
  - `SharedPreferences`를 통해 오버레이 상태 플래그 저장 및 불러오기
  - DragHandler 연동을 위해 TouchListener 생성
  - `createOverlay()`, `showOverlay()`, `hideOverlay()`, `cleanupOverlay()` 등 핵심 로직 구성

## 4. OverlayViewModel.kt
- 역할: UI와 서비스 사이의 상태 동기화, 오버레이 상태 관리 (MVVM 구조)
- 핵심 포인트:
  - `LiveData<Int>`를 통해 오버레이 플래그 UI에 반영
  - 서비스 바인딩 및 언바인딩 처리
  - `setOverlayFlag()`, `toggleOverlayFlag()` 로직으로 사용자 제어 반영

## 5. DynamicApplication.kt
- 역할: 앱 전역에서 오버레이 서비스를 제어할 수 있는 Application 클래스
- 핵심 포인트:
  - `startOverlayService()`, `stopOverlayService()` 제공
  - `AndroidManifest.xml`에 `android:name=".DynamicApplication"`으로 등록

## 6. MainActivity.kt
- 역할: 사용자 인터페이스, 오버레이 권한 요청, 플래그 제어
- 핵심 포인트:
  - `Settings.canDrawOverlays()`로 권한 체크
  - `ActivityResultContracts.StartActivityForResult()`로 권한 요청
  - ViewModel의 `overlayFlag.observe()`를 통해 UI 동기화
  - 권한 없을 경우 권한 요청 안내 및 버튼 비활성화 처리

## 7. overlay_layout.xml
- 역할: 오버레이로 띄울 View의 XML 레이아웃 정의
- 구성 요소:
  - 상단 여백(Space), 로고(ImageView), 텍스트(TextView 3개), 하단 여백(Space)
  - 적절한 여백, 정렬, 텍스트 스타일 적용됨

## 8. AndroidManifest.xml
- 핵심 설정:
  - `<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />`
  - `DynamicApplication` 등록
  - `OverlayService` 등록 (`exported="false"`, 백그라운드 전용)

---





# 전체 실행 흐름

1. 앱 실행 (`MainActivity`)
   - ActionBar 숨김, 레이아웃 설정
   - 권한 확인 → 있으면 플래그 읽고 UI 반영, 없으면 권한 요청

2. 사용자가 토글 버튼 클릭
   - `OverlayViewModel.toggleOverlayFlag()` 호출
   - `SharedPreferences`에 저장, 서비스 시작 or 중지

3. 오버레이 서비스 (`OverlayService`)
   - `setOverlayFlag()` 호출 시 → `recreateOverlay()`로 새로 생성 or `cleanupOverlay()`로 제거
   - 터치 리스너 통해 DragHandler 연동
   - View visibility에 따라 실제 표시 여부 판단 및 반영

4. 오버레이 UI
   - `OverlayManager`에서 XML inflate
   - `WindowManager.LayoutParams` 적용 후 시스템 오버레이로 부착
   - 사용자 터치로 이동 가능

---

## 특징 요약

| 구성요소 | 특징 |
|----------|------|
| `Application` | 앱 전역에서 오버레이 서비스 시작/중지 가능 |
| `ViewModel` | UI와 서비스 사이 상태 동기화 |
| `Service` | 백그라운드에서 오버레이 관리, DragHandler 포함 |
| `OverlayManager` | 뷰 생성 및 WindowManager와 연결 |
| `DragHandler` | 뷰 드래그 가능하게 처리 |
| `MainActivity` | 권한 확인 및 UI 제어 |
| `overlay_layout.xml` | 보여줄 UI 구성 (로고 + 텍스트) |










----
Context란?
Context는 앱의 현재 상태에 대한 정보와 시스템 서비스 접근을 위한 인터페이스입니다.

예: startActivity(), getSystemService(), getSharedPreferences() 등등은 전부 Context가 있어야 사용할 수 있습니다.

Application Context
항목	설명
정의	앱 전체 생명주기를 가진 Context
예시	applicationContext, getApplicationContext()
수명	앱이 메모리에 있는 한 계속 유지됨
주 용도	앱 전역에서 공유되어야 하는 자원 접근 (ex. SharedPreferences, Service 시작 등)
특징	UI에 종속되지 않음 → 액티비티/뷰 정보 없음

주로 어디서 쓰나?

뷰모델이나 싱글톤 객체에서 Context가 필요한 경우

서비스 시작할 때

앱 설정(SharedPreferences)이나 DB 초기화 등

주의할 점

화면을 띄우는 작업(startActivity)에 사용하면 오류날 수 있음 (flag 필요)

Activity/Service Context
항목	설명
정의	특정 UI(Activity)나 컴포넌트(Service)의 생명주기와 함께함
예시	this, MyActivity.this, MyService.this
수명	해당 컴포넌트가 메모리에 존재할 때까지만 유효
특징	뷰 계층 및 테마 리소스 접근 가능 → UI 작업 가능

그럼 Service Context랑 Application Context는 뭐가 다르냐?
공통점
둘 다 화면(Activity)과는 별개 → UI 요소 없음

getSystemService(), startService(), bindService() 등의 시스템 자원 접근 가능

차이점
항목	Service Context (this in Service)	Application Context (getApplicationContext())
대상	해당 서비스 인스턴스	앱 전체를 대표하는 객체
수명	서비스가 죽으면 사라짐	앱이 죽을 때까지 유지됨
특징	WindowManager로 뷰 띄우는 데 사용 가능	일부 WindowManager 타입은 지원 안 됨 (Android 버전에 따라 다름)

정리해보면
Application Context는 앱 전반에 공유되는 설정/서비스/자원에 접근할 때 유용

Service Context는 시스템 창(WindowManager), Notification 등 실제 서비스 동작에 필요한 경우 사용

OverlayService.kt에서는
WindowManager로 오버레이 뷰를 띄워야 하므로 this (서비스 컨텍스트)를 넘김

OverlayViewModel.kt에서는
application.applicationContext를 사용해서 UI와 무관하게 SharedPreferences, 서비스 시작 등을 안전하게 처리






LiveData를 굳이 사용하는 이유는 단순히 값을 담는 변수나 State가 아닌, UI와 연동되는 데이터 흐름을 관리하고, 생명주기를 안전하게 고려한 관찰 가능한 데이터 구조이기 때문이에요.

LiveData를 쓰는 이유 요약
이유	설명
1. 생명주기 자동 고려	Activity/Fragment가 onStart~onStop 동안만 데이터를 observe함 → 메모리 누수 방지
2. UI 자동 갱신	LiveData 값이 변경되면, 자동으로 UI에 반영됨 (observe 하고 있으면)
3. 상태 저장/복원 용이	ViewModel과 함께 쓰면, 화면 회전에도 상태 유지
4. 상태 공유 용이	ViewModel에서 관리되므로 여러 Fragment 간 데이터 공유 가능
5. UI-비즈니스 로직 분리	ViewModel에서 상태만 관리하고, UI는 관찰만 하도록 분리 가능

이 코드에서 LiveData를 쓰는 이유

private val _overlayFlag = MutableLiveData<Int>()
val overlayFlag: LiveData<Int> = _overlayFlag
여기서의 의미:
_overlayFlag: 내부에서 값을 변경할 수 있는 MutableLiveData

overlayFlag: 외부에서는 읽기 전용으로만 접근 가능한 LiveData

➡ 이 구조를 쓰는 이유는:

1. UI(View)가 플래그 상태를 자동으로 감지
Activity나 Fragment에서 overlayFlag.observe(...)를 하면, 값이 바뀔 때마다 자동으로 UI가 업데이트될 수 있음

2. 직접 ViewModel이 UI를 변경하지 않음
MVVM 패턴에서 UI는 오직 데이터를 관찰만 하고, ViewModel은 상태만 담당

이 구조는 유지보수성과 테스트성을 높여줌


안 쓴다면?

var overlayFlag = 0
이렇게 하면 직접 setText, invalidate, notifyDataSetChanged 같은 수동 갱신 필요

생명주기 이벤트에 맞춰 null되거나, 메모리 누수가 생길 위험 있음

결론
LiveData는 단순히 상태 저장을 넘어서,
"UI 생명주기에 안전하고, 데이터 변화에 반응적으로 대응할 수 있는 구조"를 만드는 데 필수적인 컴포넌트입니다.

실제 앱에서는 화면 전환, 회전, 백그라운드/포그라운드 등 다양한 상태 변화가 발생하므로, LiveData를 쓰는 게 안정성과 확장성 모두에 이득이에요.







바인딩(Binding)은 두 개의 컴포넌트를 서로 연결하여 데이터나 기능을 공유할 수 있도록 묶는 작업을 의미합니다. Android에서 말하는 "바인딩"은 문맥에 따라 다르게 쓰일 수 있지만, 지금까지 대화에서의 Service 바인딩을 중심으로 설명해볼게요.

서비스 바인딩이란?
Activity나 ViewModel이 Service와 연결하여 직접 내부 메서드를 호출할 수 있도록 연결하는 과정을 말합니다.

왜 바인딩이 필요할까?
Android의 Service는 보통 백그라운드에서 돌아가면서 무언가를 처리하죠.
그런데 Service에 데이터를 보내거나, Service 내부 메서드를 호출하려면 어떻게 해야 할까요?

→ 그냥은 안 됩니다.
→ bindService()를 호출해서 바인딩해야, 해당 Service의 인스턴스를 가져와서 메서드도 호출하고 상태도 읽을 수 있어요.

코드 흐름 요약 (이전 코드 기준)
// ViewModel에서
context.bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE)
bindService() 호출 → OS가 OverlayService를 시작 또는 연결

연결되면 serviceConnection.onServiceConnected() 호출됨

여기서 받은 IBinder 객체를 OverlayService.LocalBinder로 캐스팅

binder.getService()로 실제 OverlayService 인스턴스를 가져옴

이제 ViewModel에서 overlayService?.setOverlayFlag(...)처럼 직접 메서드 호출 가능

Binding vs Starting
구분	설명
startService()	단순히 서비스 시작 (기능만 수행, 통신 불가)
bindService()	서비스와 연결 (메서드 호출 등 양방향 통신 가능)
둘 다 사용 가능	예: 먼저 startService()로 시작하고, 필요할 때 bindService()로 접근 가능

정리
바인딩이란?
→ Service와 Activity, ViewModel 등이 연결되어 직접 메서드를 호출하거나 상태를 주고받을 수 있게 만드는 연결 과정이에요.
→ bindService()로 시작하고, ServiceConnection을 통해 실제 인스턴스를 받아오는 방식으로 사용합니다.
→ 바인딩을 하지 않으면, 그 Service는 단순한 백그라운드 작업만 할 수 있고, 외부에서 직접 조작은 불가능합니다.








바로 아래 코드였지:

// 연결 끊겼을 때 호출
override fun onServiceDisconnected(name: ComponentName?) {
    overlayService = null
    isServiceBound = false
}
이 메서드는 언제 호출될까?
이건 bindService()로 연결한 서비스가 예기치 않게 끊겼을 때 호출되는 콜백 함수야.

예를 들어:

서비스 프로세스가 죽었거나

시스템이 강제로 종료했거나

어떤 이유로든 연결이 끊겼을 때

이 onServiceDisconnected()가 호출돼.

코드 한 줄씩 설명

overlayService = null
지금까지 서비스 인스턴스를 참조하고 있었던 overlayService를 더 이상 사용할 수 없도록 null로 초기화해.

즉, "더 이상 연결되어 있지 않다" 라는 걸 명확히 해주기 위한 처리야.


isServiceBound = false
바인딩 상태를 나타내는 플래그를 false로 바꿔.

즉, 이제 서비스와 연결되어 있지 않다는 걸 상태로 저장해서, 이후에 다시 바인딩할 수 있게 하거나, 중복된 작업을 피하려는 목적이야.

왜 이게 중요할까?
서비스는 시스템 자원이라, 예기치 않게 죽는 경우도 있고, 네가 직접 해제하지 않았어도 연결이 풀릴 수 있어.
이때도 앱이 정상적으로 동작하기 위해, 이처럼 연결 상태를 추적해서 안정적으로 상태를 관리하는 거야.

요약 정리
역할	설명
onServiceDisconnected	서비스와의 연결이 시스템적으로 끊겼을 때 자동 호출
overlayService = null	더 이상 유효하지 않은 서비스 참조 제거
isServiceBound = false	바인딩 상태 플래그를 false로 바꿔 재연결 판단에 활용

혹시 “왜 수동으로 끊은 게 아닌데도 onServiceDisconnected가 호출돼?”가 궁금하다면,
그건 시스템이 Service를 강제로 종료했기 때문이야. 바인딩된 컴포넌트에게 알려주기 위해 이 콜백을 호출해주는 거지.










아주 좋은 질문이야. context.bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE) 이 코드의 의미와 작동 원리를 아래처럼 아주 자세히 설명해줄게:

bindService란?
bindService()는 앱이 특정 서비스(예: OverlayService)와 연결(bond)해서 그 기능을 직접 호출할 수 있게 해주는 메서드야.

즉, 서비스와 "직접적인 통신"을 위한 연결을 요청하는 거야.
startService()가 그냥 실행만 시키는 거라면, bindService()는 실제 객체에 접근해서 메서드를 호출할 수 있게 해줌.

함수 인자 분석
kotlin
context.bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE)
인자	설명
intent	연결할 서비스의 정보를 담고 있음. 보통 Intent(context, OverlayService::class.java) 형태
serviceConnection	서비스와 연결되었을 때(또는 끊겼을 때) 무엇을 할지 정의한 콜백 객체 (onServiceConnected, onServiceDisconnected)
Context.BIND_AUTO_CREATE	서비스가 아직 실행 중이 아니면 자동으로 실행시켜줌
이 플래그가 없으면, 서비스가 안 켜져 있으면 연결 안 됨

예시 없이 설명하면 어렵지? 그럼 예를 들어볼게
1. BIND_AUTO_CREATE가 있을 때
kotlin
// 서비스가 꺼져있어도 자동으로 실행 후 바인딩 시도
bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE)
서비스가 실행 중이 아니면 자동으로 startService() 한 것처럼 시작시킴

그리고 onServiceConnected() 호출됨 → 우리는 여기서 service as? OverlayService.LocalBinder 통해 인스턴스 받아옴

2. BIND_AUTO_CREATE가 없으면?
kotlin
bindService(intent, serviceConnection, 0)
서비스가 이미 실행 중일 때만 바인딩 시도

실행 중이 아니면 바인딩 실패 → onServiceConnected()도 안 불림

직접 startService()로 먼저 실행해야 함

그래서 이 코드가 하는 일
kotlin
context.bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE)
OverlayService를 실행한 후,

serviceConnection을 통해 해당 서비스 객체와 바인딩

그 결과 → onServiceConnected()에서 OverlayService의 인스턴스를 얻고, 직접 메서드 호출 가능

요약 정리
구분	의미
bindService()	서비스와 바인딩 → 인스턴스 접근 가능
serviceConnection	연결/해제 시 행동 정의
BIND_AUTO_CREATE	실행 중이 아니면 자동 실행 후 바인딩 시도

필요하다면 실제로 어떻게 overlayService?.setOverlayFlag(flag) 같은 메서드 호출이 가능한지도 보여줄게.












"외부에서 호출"이란 표현은 그 함수가 클래스 내부에서만 쓰이는 게 아니라, 클래스 외부(다른 클래스, 컴포넌트, 예: ViewModel, Activity 등)에서 사용할 수 있게 공개(public) 되어 있다는 뜻입니다.

예를 들어 startOverlayService()는 DynamicApplication 클래스 안에 있지만 public으로 정의되어 있으므로 외부에서도 아래처럼 호출할 수 있습니다:

kotlin
복사
편집
val app = application as DynamicApplication
app.startOverlayService()
즉,

DynamicApplication 안에서만 사용하는 게 아니라

OverlayViewModel 또는 다른 컴포넌트에서 오버레이 서비스를 켜거나 끌 수 있게 만든 공개 함수입니다.

왜 필요할까?
오버레이는 앱 전체에서 제어되어야 하니까

ViewModel이나 Activity에서 간접적으로 앱 전역(Application)의 상태나 서비스를 제어하려고 DynamicApplication을 매개로 접근하는 거예요.








private val viewModel: OverlayViewModel by viewModels()
는 OverlayViewModel을 현재 Activity에 바인딩(bind)하여 사용하는 코드이고, Kotlin의 위임 프로퍼티(delegate property) 를 통해 ViewModel을 자동으로 생성 및 제공받는 방식입니다.

하나씩 분해해서 설명하면:

OverlayViewModel
당신이 만든 ViewModel 클래스입니다.
오버레이 상태(플래그), 서비스 바인딩, 권한 여부 등을 UI와 분리된 채로 처리합니다.
즉, UI(Activity)는 이 ViewModel에 명령만 내리면 되고, 내부 상태나 복잡한 로직은 ViewModel이 관리합니다.

by viewModels()
viewModels()는 Android KTX가 제공하는 ViewModel 위임 함수입니다.
Activity의 생명주기에 맞춰 ViewModel 인스턴스를 제공해 줍니다.
처음 호출 시 ViewModel을 자동으로 생성하고, 이후에는 기존 인스턴스를 재사용합니다.
이 ViewModel은 내부적으로 ViewModelProvider를 통해 관리됩니다.

private val viewModel: OverlayViewModel
viewModel이라는 참조 변수를 선언하고,
OverlayViewModel 타입의 인스턴스를 가져와서 저장합니다.
Activity와 ViewModel 사이의 연결 통로가 되는 변수입니다.
비유적으로 설명하면

by viewModels()는 "이 액티비티에 맞는 ViewModel을 하나 자동으로 줘" 라는 뜻이고,
private val viewModel: OverlayViewModel은 "그걸 viewModel이라는 이름으로 저장해서 계속 쓰겠다"는 의미입니다.
왜 중요한가?

ViewModel은 Activity보다 오래 살아남을 수 있고, 화면 회전(예: 가로↔세로) 시에도 데이터 유지가 가능합니다.
즉, onCreate()가 다시 호출돼도 ViewModel은 그대로 살아 있으므로 상태가 유지됩니다.
필요하다면 viewModelProvider 방식과 비교도 해줄게요.

이 코드가 의미하는 것

private val viewModel: OverlayViewModel by viewModels()
이건 ViewModel을 현재 Activity(MainActivity)에 연결해서 쓰는 코드예요.

ViewModel이 뭐야?

ViewModel은 이렇게 생각하면 됩니다:

"Activity나 Fragment가 죽어도 데이터를 유지해주는 저장소이자,
UI 로직과 분리되어 있는 백그라운드 도우미."
즉,

Activity는 UI를 보여주는 곳
ViewModel은 상태를 저장하거나 계산을 담당하는 곳
예를 들어, 오버레이 상태(flag)가 켜졌는지 꺼졌는지 계속 기억하고 있어야 할 때
Activity가 아니라 ViewModel에 저장하면, Activity가 껐다 켜져도 값이 유지돼요.

그럼 by viewModels()는?

이건 코틀린 문법 중 하나로,
"이 ViewModel을 나 대신 알아서 만들고 관리해줘" 라는 뜻이에요.

private val viewModel: OverlayViewModel by viewModels()
이 코드는 실제로는 이런 걸 하는 거예요:

OverlayViewModel 객체를 Activity 생명주기에 맞춰 자동 생성
한 번만 생성해서 회전해도 유지
viewModel이라는 이름으로 쓰게 해줌
왜 굳이 ViewModel을 써야 해?

1. 화면 회전(세로 ↔ 가로)해도 데이터 안 날아가게 하려고
Activity는 화면 회전하면 onCreate()부터 다시 실행됨 → 데이터 날아감
ViewModel에 있으면 날아가지 않음
2. UI 로직과 상태 관리를 분리하려고
버튼 눌러서 flag 켜고 끄고 → 그 상태를 ViewModel이 관리
Activity는 "켜줘", "꺼줘"라고만 지시하면 됨
3. Lifecycle-aware (수명 자동 관리)
더 이상 안 쓰면 자동으로 정리됨
결론

항목	의미
OverlayViewModel	오버레이 상태를 저장하고 관리하는 클래스
by viewModels()	ViewModel을 자동으로 생성하고 바인딩해주는 코틀린 위임 문법
왜 씀?	UI와 데이터/상태를 분리하고, 회전해도 값 유지하고, 수명도 자동 관리하려고
필요하면 LiveData랑 같이 쓰는 이유도 이어서 설명해드릴게요!









전체 코드 설명

private val permissionLauncher = registerForActivityResult(
    ActivityResultContracts.StartActivityForResult()
) { _ ->
    // 권한 설정 후 UI 상태 재확인
    updateUIBasedOnPermission()
}
하나씩 아주 자세히 풀어보면

1. registerForActivityResult(...)
ActivityResult API를 사용해서 다른 액티비티를 실행하고,
그 결과를 받아오는 메커니즘입니다.
예전에는 startActivityForResult()를 썼는데 지금은 이 방법이 최신 방식이에요.
2. ActivityResultContracts.StartActivityForResult()
권한 설정을 위해 다른 설정 화면으로 이동할 때 사용해요.
즉, Settings.ACTION_MANAGE_OVERLAY_PERMISSION 같은 인텐트를 실행해서
사용자가 "권한 허용"을 누를 수 있도록 설정 창을 열어주는 역할입니다.
3. { _ -> updateUIBasedOnPermission() }
사용자가 설정 창에서 돌아오면 실행되는 콜백 함수예요.
_ 는 결과값을 사용하지 않겠다는 의미고,
updateUIBasedOnPermission() 을 호출해서,
→ 지금 권한을 허용했는지 다시 검사하고 UI를 최신 상태로 업데이트해요.
정리: 이 코드의 역할

단계	설명
사용자에게 오버레이 권한이 없는 경우	requestOverlayPermission()에서 인텐트를 실행함
설정창으로 이동	permissionLauncher.launch(intent)
설정 후 다시 돌아오면	위 registerForActivityResult 코드가 실행됨
권한 허용 여부 다시 확인	updateUIBasedOnPermission() 호출











좋은 질문이에요. 이 코드는 오버레이 권한이 있는지 확인하기 위한 로직인데, 안드로이드 버전마다 권한 처리 방식이 다르기 때문에 이렇게 작성된 거예요. 하나씩 설명드릴게요.

코드 설명
kotlin
private fun hasOverlayPermission(): Boolean {
    return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
        Settings.canDrawOverlays(this)
    } else {
        true
    }
}
Build.VERSION.SDK_INT >= Build.VERSION_CODES.M이란?
Android 6.0 (Marshmallow, API 23) 이상인지 확인하는 조건입니다.

Android 6.0부터 사용자가 직접 오버레이 권한을 허용해야만 앱이 다른 앱 위에 뷰를 띄울 수 있어요.

따라서 이 조건이 true면, 시스템 API인 Settings.canDrawOverlays(this)로 직접 권한을 체크해야 해요.

Settings.canDrawOverlays(this)란?
이 메서드는 앱이 다른 앱 위에 뷰를 그릴 수 있는지 확인해 줍니다.

즉, 사용자가 "다른 앱 위에 그리기" 권한을 허용했는지 여부를 알려줘요.

허용했으면 true, 아니면 false가 반환돼요.

else -> true는 왜 있나?
Android 6.0 미만 (예: Lollipop, KitKat)에서는 오버레이 권한이라는 개념 자체가 필요 없었어요.

시스템이 기본적으로 오버레이 허용 상태라서, 그냥 true를 리턴하면 됩니다.

요약
버전	권한 체크 방법
Android 6.0 이상	Settings.canDrawOverlays() 호출 필요
Android 6.0 미만	항상 true (권한 없이도 가능)

이렇게 버전 호환성을 고려해서 분기 처리를 하는 거예요.

필요하면 이와 연결되는 requestOverlayPermission() 동작도 설명드릴 수 있어요.






android:name=".DynamicApplication" 이게 필요한 이유는 앱 실행 시 커스텀 Application 클래스를 사용하기 위해서입니다.

아래에 정리해드릴게요:

기본 개념: Application 클래스란?
안드로이드 앱이 시작될 때 가장 먼저 실행되는 클래스입니다.

앱 전체의 생명주기를 관리하며, 전역 상태 저장, 서비스 시작, 초기 설정 등을 담당합니다.

원래는 Application 클래스를 직접 사용하지만, 필요하면 이를 상속해서 커스텀 클래스를 만들 수 있습니다.

왜 android:name으로 명시해야 할까?
AndroidManifest.xml에서 android:name으로 커스텀 클래스를 등록하지 않으면, 안드로이드는 기본 Application 클래스를 사용합니다.

따라서 우리가 만든 DynamicApplication.kt가 실행되도록 하려면 꼭 아래처럼 등록해야 합니다:

xml
복사
편집
<application
    android:name=".DynamicApplication"
    ... >
이 앱에서의 실제 사용 이유는?
DynamicApplication.kt 내부에서 아래 같은 전역 기능을 수행하고 있죠:


fun startOverlayService()
fun stopOverlayService()
이것은 앱 전역에서 언제든지 오버레이 서비스를 시작/중지할 수 있게 하기 위함이에요.
즉, MainActivity, ViewModel, Service 등 여러 컴포넌트에서 이 Application 객체를 참조해서 오버레이를 제어하려면 반드시 등록이 필요합니다.

android:name=".DynamicApplication"을 등록하지 않으면 앱이 DynamicApplication.kt를 인식하지 못하므로,
앱 시작 시 우리가 만든 전역 설정, 서비스 제어 등의 기능이 동작하지 않게 됩니다.








맞아요, 눈에 보이는 UI의 시작은 MainActivity부터입니다.
하지만 안드로이드 앱의 진짜 시작 지점은 Application 클래스, 즉 이 경우에는 DynamicApplication이에요.

실행 순서 정리
앱 프로세스 생성

Application 클래스 (DynamicApplication)의 onCreate() 실행

그 다음에 MainActivity가 실행됨 (onCreate() → onStart() 등 생명주기)

왜 Application이 먼저 실행될까?
안드로이드 프레임워크는 앱이 실행될 때 전체 앱 상태를 관리하기 위한 Application 객체를 먼저 초기화합니다.

Application 객체는 앱 전체에서 공유되는 싱글톤이고,

여기에 전역 설정(예: 서비스 상태 추적, 전역 변수 초기화 등)을 해두면, 앱 내 모든 컴포넌트(Activity, Service 등)에서 재사용할 수 있어요.



class DynamicApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        Log.d("TAG", "앱이 실행되자마자 제일 먼저 실행됨")
    }
}
이 로그는 MainActivity보다 먼저 찍힙니다.

MainActivity는 UI의 진입점이고,

DynamicApplication은 앱 전체의 진입점(전역 컨트롤 타워)입니다.

둘 다 중요하지만 실제 앱의 최초 진입은 Application이라는 점이 핵심입니다.









습니다. 정확히 말하면 다음과 같은 실행 흐름을 따릅니다:

안드로이드 앱 실행 흐름 요약
앱 프로세스 시작
사용자가 앱 아이콘을 터치하면, 안드로이드 시스템이 먼저 앱 프로세스를 생성합니다.

Application 클래스 인스턴스화 + onCreate() 실행
AndroidManifest.xml에서 등록된 android:name=".DynamicApplication"에 따라,
시스템은 DynamicApplication 객체를 생성하고 onCreate()를 호출합니다.
→ 여기서 앱 전역 초기화 작업을 수행할 수 있습니다.

MainActivity 시작
intent-filter에 MAIN과 LAUNCHER가 선언된 MainActivity가 자동으로 호출됩니다.
→ 이때 MainActivity의 onCreate() → onStart() → onResume() 순으로 호출되며 화면이 표시됩니다.


[앱 아이콘 클릭]
      ↓
[앱 프로세스 생성]
      ↓
[DynamicApplication.onCreate()]
      ↓
[MainActivity 생성 → onCreate() 호출 → 화면 표시]

DynamicApplication은 앱의 "진짜 시작점"이고,

MainActivity는 유저가 처음 보게 되는 "UI 진입점"입니다.

즉, DynamicApplication 실행 후 바로 MainActivity로 흐름이 이어지는 것이 맞습니다.









네, 이 프로젝트 구조는 MVVM (Model-View-ViewModel) 아키텍처 패턴을 기본적으로 따르고 있습니다. 구조적으로 보면 다음과 같이 역할이 나뉘어 있어요:

MVVM 구조로 본 구성
역할	클래스	설명
Model	SharedPreferences 또는 OverlayService 내부 상태	오버레이 플래그 등의 앱 상태 데이터를 담당합니다. DB나 API는 없지만, 상태 저장 기능을 SharedPreferences로 대체하고 있어 Model 역할을 수행합니다.
ViewModel	OverlayViewModel	View와 Model 사이에서 데이터를 연결하고, LiveData를 통해 UI 업데이트를 반영합니다. 서비스 바인딩 등 비즈니스 로직도 일부 담당합니다.
View	MainActivity, overlay_layout.xml	사용자에게 보여지는 UI입니다. ViewModel을 관찰(observe)하여 UI를 동적으로 업데이트합니다. 버튼 클릭 시 ViewModel에게 명령을 전달합니다.

왜 MVVM이라고 할 수 있나?
ViewModel이 UI를 직접 변경하지 않고, LiveData를 통해 UI가 스스로 반응하게 함

MainActivity는 오직 UI 처리에만 집중하고, 상태 변경은 OverlayViewModel에게 위임함

ViewModel은 서비스와 연결하거나 SharedPreferences 값을 읽는 비즈니스 로직을 담당함

완전한 MVVM은 아니지만…
이 구조는 간단한 MVVM 형태로, Repository 계층이나 Model 클래스를 명확히 나누진 않았어요.

하지만 View ↔ ViewModel 분리, 데이터 관찰, 상태 기반 UI 업데이트라는 MVVM의 핵심 흐름은 잘 반영되어 있습니다.

필요하면 Repository 계층까지 확장해 완전한 MVVM 구조로 리팩토링하는 방법도 알려줄 수 있어요. 원할 경우 말씀해주세요.







DragHandler에서 터치 전에 배경 상태(originalBackground)를 저장하는 이유는 드래그 시 일시적으로 배경을 바꿨다가, 드래그가 끝난 후 원래 배경으로 되돌리기 위해서입니다.

왜 배경을 바꾸고 되돌릴까?
드래그가 시작되면 일반적으로 사용자에게 피드백을 주기 위해 UI에 변화를 줍니다. 예를 들면:

색깔을 살짝 진하게 바꾸거나

테두리를 강조하거나

그림자를 추가하거나

그런데 이렇게 바뀐 상태를 드래그 끝나고 복원하려면, 이전 상태를 기억해야 하겠죠? 그래서:

kotlin
private var originalBackground: GradientDrawable? = null
이렇게 드래그 전 배경을 복사해 저장해두는 거예요.

언제 쓰이냐면?
예시 흐름은 다음과 같아요:

사용자가 뷰를 터치하면 MotionEvent.ACTION_DOWN 발생

originalBackground = view.background as? GradientDrawable 로 기존 배경 저장

view.setBackground(...) 으로 드래그 중 시각 효과 제공

MotionEvent.ACTION_UP 또는 ACTION_CANCEL 시

원래 배경으로 복원: view.background = originalBackground


목적: 드래그 중 UI 피드백 제공 + 드래그 종료 시 원래 상태로 복원

안 하면?: 드래그 이후에 UI가 변경된 채로 남아서 불안정해 보이거나 시각적으로 어색해질 수 있음









좋은 질문이에요. 이 라인에서 as?가 여러 번 등장하는 이유는 형 변환(casting)을 안전하게 단계적으로 수행하기 위해서입니다. 이 코드를 해석하면 다음과 같은 과정으로 이루어져 있어요:

원래 코드
kotlin
originalBackground = (overlayView.background as? GradientDrawable)?.constantState?.newDrawable() as? GradientDrawable
한 단계씩 나눠서 설명
1. overlayView.background as? GradientDrawable
overlayView.background는 Drawable 타입입니다.

근데 우리는 GradientDrawable만 다루고 싶기 때문에 안전한 다운캐스팅 시도:

kotlin
val gradient = overlayView.background as? GradientDrawable
만약 다른 Drawable이라면 null이 됩니다 (앱이 죽지 않음).

2. gradient.constantState?.newDrawable()
Drawable 객체는 내부 상태(ConstantState)를 복제해서 새로운 Drawable을 만들 수 있어요.

newDrawable()은 원본을 직접 수정하지 않고 복사본을 만들기 위한 것입니다.

3. (gradient.constantState?.newDrawable()) as? GradientDrawable
newDrawable()이 반환하는 타입은 일반적으로 Drawable입니다.

다시 한번 안전하게 GradientDrawable로 캐스팅해야 하므로 as? GradientDrawable 사용.

정리: 왜 as?가 2번이나 필요하냐?
처음에는 Drawable을 GradientDrawable로 바꾸기 위해

두 번째는 newDrawable() 결과(일반 Drawable)를 GradientDrawable로 다시 바꾸기 위해

중간에 하나라도 실패하면 null이 되어 앱이 죽지 않도록 안전한 형 변환(as?)을 사용한 거예요.

참고: 전체 흐름 예시
kotlin
val background = overlayView.background
val gradient = background as? GradientDrawable
val cloned = gradient?.constantState?.newDrawable() as? GradientDrawable
originalBackground = cloned
결과적으로 originalBackground에는 현재 배경의 안전한 복제본이 들어가게 됩니다.









GradientDrawable이어야 하는 이유는 DragHandler에서 오버레이의 배경을 동적으로 조작(예: 터치 중 색상 변경 등) 하기 위해서입니다. GradientDrawable은 Android에서 모양(shape), 색상(color), 테두리(border), 코너 반경(corner radius) 등을 프로그래밍으로 제어할 수 있는 Drawable이기 때문이에요.

정리하면:
1. overlayView.background이 반드시 GradientDrawable이어야 하는 이유
색상 변경을 하려면 setColor(int) 같은 메서드가 필요함

이 메서드는 Drawable 인터페이스에는 없고, GradientDrawable에만 있음

kotlin
(overlayView.background as GradientDrawable).setColor(Color.RED)
예시: 만약 GradientDrawable이 아니라면?

kotlin
val bg = overlayView.background
bg.setColor(Color.RED) // 컴파일 에러: Drawable에는 setColor가 없음!
→ 이런 이유로 Drawable을 무조건 GradientDrawable로 변환(cast)해서 작업해야 함

실무에서의 목적
터치 전 배경을 저장하고

터치 중에는 배경 색 변경

터치가 끝나면 원래 배경으로 복원

이 흐름을 위해서는 배경이 GradientDrawable이어야 정상 동작합니다.

보완 팁
만약 배경이 GradientDrawable이 아닐 수도 있다면, as?를 사용해 안전하게 처리하고,

아니면 앱을 디자인할 때부터 XML에서 배경을 반드시 GradientDrawable로 설정해야 함

xml
<!-- overlay_layout.xml -->
<LinearLayout
    ...
    android:background="@drawable/overlay_background" />
xml
<!-- res/drawable/overlay_background.xml -->
<gradient xmlns:android="http://schemas.android.com/apk/res/android"
    android:angle="0"
    android:startColor="#FFFFFF"
    android:endColor="#EEEEEE"
    android:type="linear" />
이렇게 해두면 .background는 항상 GradientDrawable이 되므로 캐스팅 문제가 줄어듭니다.











좋습니다. 아래에 이 함수 navigateToMainApp()을 줄 단위로 아주 자세히 설명해드릴게요. 코드 블록으로 바로 복사해서 쓸 수 있게 마크다운 형식으로 구성했습니다:

kotlin
private fun navigateToMainApp() {
private 접근제한자이므로 이 함수는 이 클래스 내에서만 호출 가능

이름 그대로 "메인 앱으로 이동(navigate)"하는 역할을 수행

kotlin
    try {
예외가 발생할 수 있으므로 try-catch 블록 사용

예: Intent가 잘못되었거나 Context가 유효하지 않을 때 앱이 크래시 나지 않도록 처리

kotlin
        val intent = Intent(context, MainActivity::class.java).apply {
Intent 객체 생성: context를 기반으로 MainActivity를 실행하려고 함

apply { ... } 블록은 intent 객체에 플래그를 설정하기 위한 Kotlin 문법 (스코프 함수)

kotlin
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or // 새 태스크(Task)로 시작
                    Intent.FLAG_ACTIVITY_CLEAR_TOP or // 기존 액티비티 스택에 있으면 제거
                    Intent.FLAG_ACTIVITY_SINGLE_TOP // 동일한 액티비티가 맨 위에 있으면 재사용
세 가지 플래그를 OR 연산자로 조합해 설정합니다:

플래그	설명
FLAG_ACTIVITY_NEW_TASK	context가 Activity가 아닌 경우 필수 (예: Service, Application)에서 Activity를 실행할 때 필요
FLAG_ACTIVITY_CLEAR_TOP	이미 실행 중인 MainActivity가 스택에 있으면 해당 액티비티 위에 있는 모든 액티비티를 제거하고 그 위에 재시작
FLAG_ACTIVITY_SINGLE_TOP	이미 스택 맨 위에 MainActivity가 있다면 새로 생성하지 않고 기존 것을 재사용 (onNewIntent 호출됨)

즉, 앱이 백그라운드에 있든, 여러 액티비티가 열려 있든 상관없이 MainActivity만 하나 있도록 만드는 안정적인 방법

kotlin
        }
apply 블록 종료

kotlin
        context.startActivity(intent)
위에서 설정한 Intent를 실제로 실행

context가 Application, Service 등 Activity가 아닐 수 있으므로 NEW_TASK 플래그가 필수

kotlin
    } catch (e: Exception) {
예외 발생 시 catch 블록으로 이동

kotlin
        e.printStackTrace()
콘솔에 에러 로그 출력

예: context가 null이거나 MainActivity가 AndroidManifest에 등록되어 있지 않으면 예외 발생 가능

kotlin
    }
}
함수 종료

요약
navigateToMainApp() 함수는 어느 컴포넌트에서든(MainActivity가 아닌 곳 포함) 앱의 메인 화면으로 안정적으로 이동할 수 있도록 다음을 보장합니다:

백그라운드에서 실행해도 크래시 방지 (NEW_TASK)

여러 액티비티가 떠 있어도 정리 (CLEAR_TOP)

이미 실행 중이면 새로 생성하지 않고 재사용 (SINGLE_TOP)

이 패턴은 Service나 BroadcastReceiver에서 Activity를 실행할 때 자주 사용되는 안전한 방법입니다.












context는 안드로이드에서 앱의 현재 상태나 환경에 대한 정보를 담고 있는 객체예요. 쉽게 말해:

"Context는 안드로이드 시스템에게 지금 어디서 무엇을 하고 있는지 알려주는 창구"입니다.

예를 들어 설명하면:
context.startActivity(...): 어디서 액티비티를 시작할지 알려줘야 하니까 context가 필요

context.getSystemService(...): 시스템 서비스 (예: WindowManager, ClipboardManager)를 가져올 때

context.getSharedPreferences(...): 앱의 설정 파일에 접근할 때

context.getResources().getString(R.string.xxx): 문자열, 색상, 이미지 등 리소스에 접근할 때

Context의 주요 종류
타입	설명	예시
Activity Context	하나의 화면(Activity)에 대한 context	this (Activity 내부)
Application Context	앱 전체에 공통된 context	applicationContext
Service Context	서비스에서 동작할 때 사용	Service 내부의 this
BroadcastReceiver Context	브로드캐스트 수신 시 자동으로 주어지는 context	onReceive의 첫 번째 매개변수

언제 어떤 Context를 써야 하나?
상황	적절한 Context
UI 요소를 직접 띄우거나, Activity 전환	Activity Context
오랜 시간 살아있는 객체가 필요하거나 앱 전체에 적용	Application Context
Service에서 Activity 실행	Application Context + Intent.FLAG_ACTIVITY_NEW_TASK 필요

예시 코드
kotlin
// 액티비티 안에서
val prefs = getSharedPreferences("prefs", Context.MODE_PRIVATE) // this도 context

// ViewModel 또는 다른 컴포넌트에서
val prefs = context.getSharedPreferences("prefs", Context.MODE_PRIVATE)
왜 중요한가?
Context를 잘못 사용하면 메모리 누수(예: 액티비티가 사라졌는데도 여전히 참조 중)나 앱 크래시가 발생할 수 있습니다.
그래서 ApplicationContext vs ActivityContext 구분이 매우 중요합니다.

필요하시면 두 개의 차이도 정리해드릴게요.











이 코드는 뷰의 좌우 여백(padding)을 설정하는 로직입니다. 한 줄씩 아주 자세하게 설명드릴게요:

```kotlin
val paddingPx = dpToPx(responsiveSizes.padding24dp.toFloat()).toInt()

markdown
복사
편집

- `responsiveSizes.padding24dp`:  
  `24dp` 크기의 패딩 값을 **기기 해상도에 맞춰 조정된 dp값**입니다.
  - 예: 기준 해상도 대비 작은 기기에서는 20dp 정도가 될 수도 있고, 큰 기기에서는 28dp가 될 수도 있어요.

- `.toFloat()`:  
  변환 함수 `dpToPx()`는 `Float` 타입을 받기 때문에 `Int → Float` 변환이 필요합니다.

- `dpToPx(...)`:  
  dp 단위를 실제 픽셀(px)로 바꾸는 함수입니다.
  - 예: `24dp → 72px` (기기의 화면 밀도에 따라 달라짐)

- `.toInt()`:  
  `dpToPx()`는 `Float`을 반환하므로, 최종적으로 `Int`로 반올림하여 사용합니다.
  - UI 요소의 크기는 정수로 설정해야 하니까요.

> **결과적으로**: `paddingPx`는 기기 해상도에 맞춘 **24dp의 px값**이 됩니다.

---

### ```kotlin
cont.setPadding(paddingPx, 0, paddingPx, 0)
cont: setPadding을 적용할 대상 View입니다 (예: LinearLayout, TextView 등)

setPadding(left, top, right, bottom) 형식입니다.

여기선 좌우(left, right)에만 패딩을 주고, 상하(top, bottom)는 0으로 설정했습니다.

결과
이 코드는 해당 뷰의 왼쪽과 오른쪽에 24dp만큼의 여백을 주는 효과를 줍니다.
dpToPx()를 통해 해상도에 맞게 조정되므로 어떤 기기에서든 일관된 시각적 여백을 보장합니다.

필요하시면 dpToPx() 함수 정의도 알려드릴게요.






결론부터 말하면 setPadding() 함수는 정수(Int) 값을 요구하기 때문입니다.

왜 Int로 변환해야 하나?
안드로이드에서 padding, margin, width, height 등 뷰의 위치나 크기를 설정하는 모든 속성은 Int 타입의 픽셀(px) 값으로 지정해야 합니다.

kotlin
fun View.setPadding(left: Int, top: Int, right: Int, bottom: Int)
이렇게 시그니처가 되어 있기 때문에, Float이나 Double 타입을 넣으면 컴파일 에러가 납니다.

그런데 왜 Float로 계산하나요?
dp → px 변환 공식이 소수점 결과를 내기 때문입니다:

kotlin
px = dp * (density)  // density는 소수점일 수 있음 (예: 2.75)
예를 들어 24dp * 2.75 = 66.0px 이 되는데, 이 값을 setPadding()에 넣으려면 Int로 바꿔야 하니까:

kotlin
66.0f.toInt() → 66
요약
setPadding()은 Int만 받기 때문에 toInt()가 필수입니다.

하지만 dpToPx()는 정확한 변환을 위해 Float으로 먼저 계산하고,

마지막에 반올림 없이 내림(toInt()) 해서 사용합니다.

필요하다면 dpToPx() 구현도 같이 보여드릴 수 있어요.









좋습니다. 지금까지 살펴본 구조를 바탕으로 이 앱에서 오버레이(Overlay) 를 구현한 전체 과정을 한 줄씩 정리해드릴게요.

🧩 오버레이(Overlay)는 어떻게 구현되었나?
1. 권한 획득
오버레이 표시를 위해 반드시 SYSTEM_ALERT_WINDOW 권한이 필요합니다.

MainActivity에서 Settings.canDrawOverlays(this)로 권한 여부를 확인하고, 없으면 ACTION_MANAGE_OVERLAY_PERMISSION 인텐트로 설정 화면을 엽니다.

2. 오버레이 뷰 정의
res/layout/overlay_layout.xml 에 오버레이로 띄울 UI 정의

LinearLayout 안에 ImageView, TextView, Space 등을 배치

디자인은 따릉이 대여 상태를 알려주는 구성

3. 오버레이를 관리하는 OverlayManager
OverlayManager.kt 클래스는 오버레이 뷰를 WindowManager에 추가하거나 제거하는 기능 담당

내부에서 WindowManager.LayoutParams 를 만들어 시스템 창 위에 떠 있도록 설정

예:

kotlin
windowManager.addView(overlayView, layoutParams)
4. DragHandler로 드래그 가능하게
DragHandler.kt 는 OnTouchListener 를 통해 사용자가 오버레이 뷰를 마우스/손가락으로 끌 수 있도록 좌표를 계산하고 이동시킵니다.

5. OverlayService에서 백그라운드 실행
OverlayService.kt 는 Service 를 상속한 백그라운드 서비스

앱이 꺼져도 유지 가능

OverlayManager 와 DragHandler 를 내부에서 생성 및 관리

오버레이를 띄우거나 숨기는 로직 실행

View.VISIBLE / View.GONE 으로 상태 제어

6. ViewModel로 서비스와 UI 연결
OverlayViewModel.kt 은 LiveData<Int> 로 오버레이 플래그(0 또는 1) 를 관리

MainActivity는 ViewModel을 관찰하며 UI를 업데이트하고 서비스와 연동

7. DynamicApplication에서 서비스 시작/중지 통제
전역 Application 클래스에서 startService() / stopService() 호출을 래핑

앱 어디서든 오버레이 서비스를 시작하거나 중지 가능

8. AndroidManifest에 권한 및 서비스 등록
SYSTEM_ALERT_WINDOW 권한 명시

OverlayService 정의

요약: 오버레이 구현 흐름
사용자에게 오버레이 권한 요청

MainActivity → ViewModel에서 flag 설정

ViewModel → DynamicApplication을 통해 OverlayService 시작

OverlayService → OverlayManager와 DragHandler로 오버레이 생성

뷰를 WindowManager에 추가 (오버레이 시스템 창)

뷰는 사용자에 의해 이동 가능하며 View.VISIBLE / View.GONE 으로 상태 변경

앱이 종료되어도 서비스가 살아 있다면 오버레이 유지됨

필요하다면 각 단계별로 코드 라인까지 추적해 드릴 수 있어요.
혹시 특정 부분이 더 궁금한가요?







좋아요. 그러면 “진짜로 뷰가 화면에 띄워지는 핵심 부분”만 아주 구체적으로 설명드릴게요. 이건 오버레이의 핵심이자, 안드로이드 시스템 창 위에 UI를 띄우는 메커니즘입니다.

뷰를 띄우는 핵심 코드: WindowManager.addView(...)
위치: OverlayManager.kt
kotlin
복사
편집
windowManager.addView(overlayView, layoutParams)
어떤 흐름으로 실행되느냐?
1. OverlayService.kt → createOverlay()
kotlin
복사
편집
val result = overlayManager?.createOverlay(touchListener)
OverlayManager의 createOverlay() 호출해서 뷰 생성 + 추가

2. OverlayManager.kt → createOverlay(...)
kotlin
fun createOverlay(touchListener: View.OnTouchListener): Pair<View, WindowManager.LayoutParams>? {
    val inflater = context.getSystemService(Context.LAYOUT_INFLATER_SERVICE) as LayoutInflater
    val view = inflater.inflate(R.layout.overlay_layout, null)  // XML을 View 객체로 변환

    val params = createLayoutParams()  // WindowManager.LayoutParams 생성

    view.setOnTouchListener(touchListener)  // DragHandler 등록

    windowManager.addView(view, params)  // 바로 이 부분에서 진짜 뷰가 화면에 띄워짐

    overlayView = view
    layoutParams = params

    return Pair(view, params)
}
핵심 구성 요소 설명
구성요소	설명
WindowManager	시스템 전체에 뷰를 띄울 수 있는 매니저. 액티비티 외부에서도 사용 가능
addView(view, params)	특정 View를 시스템 윈도우 위에 직접 추가함
view	overlay_layout.xml에서 inflate된 실제 레이아웃
params	이 뷰의 크기, 위치, 타입, 투명도 등 설정 정보를 담은 파라미터

WindowManager.LayoutParams 예시
kotlin
val params = WindowManager.LayoutParams(
    WindowManager.LayoutParams.WRAP_CONTENT,
    WindowManager.LayoutParams.WRAP_CONTENT,
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O)
        WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY
    else
        WindowManager.LayoutParams.TYPE_PHONE,
    WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or
        WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL or
        WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH,
    PixelFormat.TRANSLUCENT
)
params.gravity = Gravity.TOP or Gravity.START
params.x = 100
params.y = 100
TYPE_APPLICATION_OVERLAY: 다른 앱 위에 표시할 수 있는 유형 (API 26 이상)

FLAG_NOT_FOCUSABLE: 포커스를 받지 않음 (키보드 입력 등 차단)

PixelFormat.TRANSLUCENT: 배경을 투명하게 처리

시각적으로 어떻게 보이냐?
이 뷰는 앱의 액티비티가 아니더라도 화면에 계속 떠 있게 됨.

예를 들어 홈 화면이나 다른 앱을 켜도, 이 뷰는 계속 남아 있게 됨.

이는 일종의 '시스템 오버레이 레이어'에 뷰를 추가했기 때문입니다.

요약
오버레이에서 진짜로 뷰를 띄우는 코드 핵심은 이 한 줄입니다:

kotlin
windowManager.addView(view, layoutParams)
이 줄은 Android 시스템 레벨에서 해당 뷰를 화면 위에 그리도록 요청하는 것이며, 이 방식 덕분에 액티비티가 없어도, 앱이 백그라운드여도 UI를 띄울 수 있게 됩니다.

필요하다면 이 줄의 동작 원리를 시스템 레벨까지 파고 설명해드릴 수도 있어요. 더 궁금한 부분 있어요?











좋은 질문이에요. Android에서 API 26 미만 (Android 8.0 이하) 버전에서는 TYPE_APPLICATION_OVERLAY가 없기 때문에, 대신 TYPE_PHONE 또는 TYPE_SYSTEM_ALERT 같은 윈도우 타입을 사용해서 오버레이를 띄웁니다. 이제 그 차이와 작동 방식, 그리고 주의사항을 정리해드릴게요.

API 26 이상 vs 이하의 차이
항목	API 26 이상 (Android 8.0+)	API 25 이하 (Android 7.x 이하)
사용 타입	TYPE_APPLICATION_OVERLAY	TYPE_PHONE 또는 TYPE_SYSTEM_ALERT
권한	SYSTEM_ALERT_WINDOW 필요 + 설정 화면 허용 필요	SYSTEM_ALERT_WINDOW 필요
특징	보안 및 사용자 제어 강화됨	앱이 마음대로 띄울 수 있음 (남용 가능성 있음)
제한 사항	사용자 설정에서 명시적으로 허용해야만 작동	기본적으로 권한만 있으면 작동

코드 비교
1. API 26 이상 (Android 8.0 이상)
kotlin
복사
편집
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    layoutParams.type = WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY
}
2. API 25 이하
kotlin
복사
편집
@Suppress("DEPRECATION")
layoutParams.type = WindowManager.LayoutParams.TYPE_PHONE
이렇게 분기 처리합니다.

TYPE_PHONE이란?
TYPE_PHONE은 원래 전화 앱의 UI (예: 통화창)가 다른 앱 위에 뜨도록 만든 창 유형입니다.

하지만 개발자들이 이를 남용하면서 광고나 악성 앱들이 사용자에게 불편을 주는 데 사용되어,

Android 8.0부터는 제한되고 TYPE_APPLICATION_OVERLAY로 대체되었습니다.

사용 시 권한
AndroidManifest.xml에 반드시 다음 권한이 있어야 작동합니다:

xml
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
또한 사용자는 반드시 "다른 앱 위에 그리기 허용"을 설정 화면에서 직접 허용해주어야 합니다:

kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
    if (!Settings.canDrawOverlays(context)) {
        val intent = Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION, Uri.parse("package:${context.packageName}"))
        context.startActivity(intent)
    }
}
API 26 이하에서 오버레이가 뜨는 원리 요약
WindowManager.addView()로 실제 시스템 창 레이어에 뷰를 올림

이때 type = TYPE_PHONE으로 설정되면 안드로이드 시스템은 그 뷰를 앱 외부에 그릴 수 있도록 허용

이 모든 건 SYSTEM_ALERT_WINDOW 권한이 허용된 경우에만 동작

주의사항
Google Play 정책상 SYSTEM_ALERT_WINDOW 사용 앱은 배포 제한 또는 설명 의무가 따릅니다.

사용자의 동의를 얻지 않고 팝업을 띄우거나 UI를 가리면 제재될 수 있습니다.

필요하다면 실제 버전 체크 후 처리하는 함수 샘플도 드릴 수 있어요.
계속해서 API 23 이상, 26 이상에 대한 분기 로직이나 best practice도 알려드릴까요?











"서비스만 켜져 있으면 실행된다"는 말은, 화면(UI)이 없어도 백그라운드에서 동작이 가능하다는 의미예요. 이건 안드로이드 Service의 특징에서 나오는 이야기입니다.

아래에서 더 자세히 설명해드릴게요.

안드로이드에서 "서비스"란?
Service는 사용자가 직접 눈으로 볼 수는 없지만, 앱이 종료되거나 액티비티가 사라져도 계속 백그라운드에서 동작할 수 있는 컴포넌트입니다.

예를 들어:

음악 재생

위치 추적

오버레이 띄우기

알림 처리

같은 작업을 담당합니다.

"서비스만 켜져 있으면 오버레이가 실행된다"는 의미
예를 들어, 이런 흐름을 상상해보세요:

MainActivity에서 토글 버튼을 눌러 OverlayService를 시작함

OverlayService가 실행되면서 오버레이 뷰를 띄움 (WindowManager.addView(...))

이 시점에서 MainActivity가 꺼져도

오버레이는 그대로 화면에 떠 있고

OverlayService가 계속 실행 중이므로 유지됨

즉, 앱의 UI(Activity)는 꺼져도,
서비스가 살아 있기만 하면 오버레이는 유지되고 반응할 수 있다는 뜻입니다.

예시 흐름
plaintext
복사
편집
[앱 실행]
  ↓
[MainActivity → OverlayService 시작]
  ↓
[OverlayService 내부에서 WindowManager로 오버레이 뷰 추가]
  ↓
[MainActivity 종료해도]
     ↓
[OverlayService가 살아 있으면 오버레이 유지]
왜 중요한가?
오버레이 UI는 Activity와 독립적으로 살아야 함
→ 예: "따릉이 대여 중" 안내가 항상 떠 있어야 한다면, 사용자가 다른 앱을 쓰고 있어도 보여야 하니까요.

사용자 경험 개선
→ 앱을 완전히 닫지 않아도 알림이나 안내 기능을 유지할 수 있어요.

한마디 요약
서비스만 실행 중이면,
앱 화면이 꺼지거나 앱이 백그라운드에 있어도
오버레이는 계속 보여질 수 있다.

필요하시면 서비스가 언제 죽을 수 있는지나 포그라운드 서비스로 전환하는 방법도 알려드릴게요.










오버레이 뷰(Overlay View)의 최소/최대 크기는 _명시적으로 XML이나 코드에서 설정하지 않으면 시스템 제한에 따르거나 뷰의 내용에 따라 결정_됩니다. 현재 코드에서는 아래 방식으로 간접적으로 크기 조절이 이뤄지고 있어요:

1. XML의 기본 크기: wrap_content
xml
<LinearLayout
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
wrap_content는 자식 뷰 크기만큼만 크기를 잡는다는 의미예요.

따라서 자식 뷰들의 크기 설정이 사실상 전체 오버레이 크기의 기준이 됩니다.

2. 자식 뷰 크기 설정 (예: logoView)
kotlin
val logoParams = logo.layoutParams as? LinearLayout.LayoutParams ?: LinearLayout.LayoutParams(
    dpToPx(responsiveSizes.logoSize.toFloat()).toInt(),
    dpToPx(responsiveSizes.logoSize.toFloat()).toInt()
)
logoParams.width = dpToPx(responsiveSizes.logoSize.toFloat()).toInt()
logoParams.height = dpToPx(responsiveSizes.logoSize.toFloat()).toInt()
responsiveSizes.logoSize는 보통 40~60dp 수준으로 설정돼 있어요.

결국 이 값이 간접적인 최소/최대 크기를 결정하게 됩니다.

3. WindowManager.LayoutParams로 최종 제어
kotlin
val params = WindowManager.LayoutParams(
    WindowManager.LayoutParams.WRAP_CONTENT,
    WindowManager.LayoutParams.WRAP_CONTENT,
    ...
)
WRAP_CONTENT라서 역시 뷰 자체의 크기에 맞춰 자동 조정됨.

하지만 원한다면 다음처럼 명시적으로 최소/최대 크기를 지정할 수도 있어요:

kotlin
params.width = dpToPx(100f).toInt()
params.height = dpToPx(60f).toInt()
참고: 시스템 제한
Android는 전체 화면 너비/높이를 초과하는 Window를 제한하지는 않지만,

유저 경험 보호를 위해 너무 큰 오버레이는 경고나 권한 제한의 원인이 될 수 있어요.

따라서 보통 화면 비율의 10~30% 이내로 제한하는 게 일반적입니다.

결론: 크기 규격은 어디?
XML에서 wrap_content로 시작하고

자식 뷰의 dp 값 (예: logoSize 등) 에 의해 크기 결정

코드에서 LayoutParams의 width/height 설정을 통해 최종 조정

명시적 최소/최대값은 따로 설정된 부분은 없음, 대신 responsiveSizes로 간접 제어됨

필요하다면 minWidth, maxWidth 등을 코드나 XML에서 강제로 줄 수 있어요.
이 기능을 적용하고 싶다면 예시도 만들어드릴 수 있어요.











실제 오버레이 뷰의 화면 상 픽셀 단위 크기(width/height) 를 확인하고 싶다면, 실제 뷰가 화면에 추가된 이후 view.width, view.height 값을 로그로 찍어보는 방법이 가장 확실합니다. 하지만 주의할 점이 있습니다:

문제: view.width와 view.height는 바로 접근하면 0일 수 있음
View는 화면에 측정(measure) 되고 배치(layout) 된 이후에만 width, height 값이 정해지기 때문에, 생성 직후에는 0일 수 있습니다.

해결 방법 1: post {}를 이용한 딜레이 측정
kotlin

overlayView?.post {
    val width = overlayView?.width
    val height = overlayView?.height
    Log.d("OverlaySize", "실제 크기: ${width}px x ${height}px")
}
post {} 블록은 뷰가 레이아웃 트리에 붙은 다음 프레임에서 실행되므로, 측정된 크기를 안전하게 읽을 수 있습니다.

해결 방법 2: ViewTreeObserver를 이용한 콜백 등록
kotlin

overlayView?.viewTreeObserver?.addOnGlobalLayoutListener {
    val width = overlayView?.width
    val height = overlayView?.height
    Log.d("OverlaySize", "실제 크기: ${width}px x ${height}px")
}
뷰의 레이아웃 완료 시점에 한 번만 실행할 수 있어요.

사용 후 removeOnGlobalLayoutListener() 로 정리하는 게 좋습니다.

로그와 실제 크기가 다르게 보이는 이유는?
dpToPx()로 계산했지만 소수점 오차가 있음

padding, margin, layout_weight 등이 더해져 최종 크기가 달라질 수 있음

부모 뷰나 WindowManager.LayoutParams에서 제한되는 경우

XML 값과 코드 값이 덮어씌워지면서 혼재됨

디버깅 팁
overlayView.width / height 로그 찍기

WindowManager.LayoutParams.width, height도 같이 비교

layoutParams 값도 출력해 보기:

kotlin

val lp = overlayView?.layoutParams as? ViewGroup.LayoutParams
Log.d("LayoutParams", "layoutParams.width=${lp?.width}, layoutParams.height=${lp?.height}")
요약
측정 시점	방법	설명
가장 안전한 방법	post {}	레이아웃 이후 보장
정밀한 감지	ViewTreeObserver	레이아웃 이벤트 수신 가능
피해야 할 시점	뷰 생성 직후	width, height는 0일 가능성 높음

필요하다면 로그 찍는 샘플 코드를 전체 삽입해드릴 수 있습니다. 원하시면 알려주세요.









맞습니다, 정확하게 이해하셨어요.

요약하면:
오버레이 크기 계산은 디바이스의 "물리적인 크기"가 아니라, "현재 화면의 가로/세로 크기 (dp 단위) 비율"만 가지고 계산합니다.

좀 더 구체적으로 말하자면:
기준 화면 크기 (REFERENCE_SCREEN_WIDTH, REFERENCE_SCREEN_HEIGHT)
→ 예: 412dp x 892dp는 디자이너가 만든 UI 기준 해상도입니다.

현재 기기 화면 크기 (dp)
→ 예: Flip6은 360dp x 880dp, Fold6은 928dp x 1080dp 등입니다.

가로 비율, 세로 비율 각각 계산 → 더 작은 값 사용

kotlin
val widthRatio = currentWidth / referenceWidth
val heightRatio = currentHeight / referenceHeight
val minRatio = min(widthRatio, heightRatio)
이 minRatio는 "기준 화면보다 얼마나 작거나 큰지"를 나타내는 척도예요.

기준보다 작은 기기는 축소되고, 큰 기기는 확대됩니다.

폰의 물리 크기, 해상도, 밀도(dpi)
→ 직접적으로 사용하지 않습니다.
→ 이미 dp 단위는 해상도와 밀도를 추상화한 단위이기 때문에, DisplayMetrics로 얻은 dp 값만으로도 충분합니다.

그래서 결론은:
Galaxy Z Fold6처럼 물리적으로 큰 화면이라도 dp 기준 비율이 기준보다 크면 커지게 되고,

Galaxy Z Flip6처럼 작은 화면은 기준보다 비율이 작으니 작아지는 구조입니다.

즉, 디바이스 종류나 실제 인치 크기와는 상관없이 "UI 기준 해상도 대비 비율"만으로 크기를 결정하는 방식입니다.
이 방식은 다양한 기기에서도 UI 비율을 일정하게 유지할 수 있다는 장점이 있어요.

