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
