### 一、概要
100行代码实现recyclerview条目曝光埋点设计

### 二、设计思路
1. 条目露出来一半以上视为该条目曝光。
2. 在rv滚动过程中或者数据变更回调OnGlobalLayoutListener时，将符合条件1的条目记录在曝光列表、上传埋点集合里。
3. 滚动状态变更和OnGlobalLayoutListener回调时，且列表状态为idle状态，触发上报埋点。

### 三、容错性
1. 滑动过快时，视为未曝光
2. 数据变更时，重新检测曝光
3. 曝光过的条目，不会重复曝光

### 四、接入影响
1. 对业务代码零侵入
2. 对列表滑动体验无影响

### 五、代码实现
```kotlin
import android.graphics.Rect
import android.view.View
import androidx.recyclerview.widget.RecyclerView
import java.util.*

class RVItemExposureListener(
    private val mRecyclerView: RecyclerView,
    private val mExposureListener: IOnExposureListener?
) {
    interface IOnExposureListener {
        fun onExposure(position: Int)
        fun onUpload(exposureList: List<Int>?): Boolean
    }

    private val mExposureList: MutableList<Int> = ArrayList()
    private val mUploadList: MutableList<Int> = ArrayList()
    private var mScrollState = 0

    var isEnableExposure = true
    private var mCheckChildViewExposure = true

    private val mViewVisible = Rect()
    fun checkChildExposeStatus() {
        if (!isEnableExposure) {
            return
        }
        val length = mRecyclerView.childCount
        if (length != 0) {
            var view: View?
            for (i in 0 until length) {
                view = mRecyclerView.getChildAt(i)
                if (view != null) {
                    view.getLocalVisibleRect(mViewVisible)
                    if (mViewVisible.height() > view.height / 2 && mViewVisible.top < mRecyclerView.bottom) {
                        checkExposure(view)
                    }
                }
            }
        }
    }

    private fun checkExposure(childView: View): Boolean {
        val position = mRecyclerView.getChildAdapterPosition(childView)
        if (position < 0 || mExposureList.contains(position)) {
            return false
        }
        mExposureList.add(position)
        mUploadList.add(position)
        mExposureListener?.onExposure(position)
        return true
    }

    private fun uploadList() {
        if (mScrollState == RecyclerView.SCROLL_STATE_IDLE && mUploadList.size > 0 && mExposureListener != null) {
            val success = mExposureListener.onUpload(mUploadList)
            if (success) {
                mUploadList.clear()
            }
        }
    }

    init {
        mRecyclerView.viewTreeObserver.addOnGlobalLayoutListener {
            if (mRecyclerView.childCount == 0 || !mCheckChildViewExposure) {
                return@addOnGlobalLayoutListener
            }
            checkChildExposeStatus()
            uploadList()
            mCheckChildViewExposure = false
        }
        mRecyclerView.addOnScrollListener(object : RecyclerView.OnScrollListener() {
            override fun onScrollStateChanged(
                recyclerView: RecyclerView,
                newState: Int
            ) {
                super.onScrollStateChanged(recyclerView, newState)
                mScrollState = newState
                uploadList()
            }

            override fun onScrolled(
                recyclerView: RecyclerView,
                dx: Int,
                dy: Int
            ) {
                super.onScrolled(recyclerView, dx, dy)
                if (!isEnableExposure) {
                    return
                }

                // 大于50视为滑动过快
                if (mScrollState == RecyclerView.SCROLL_STATE_SETTLING && Math.abs(dy) > 50) {
                    return
                }
                checkChildExposeStatus()
            }
        })
    }
}

```

### 六、使用
```kotlin
RVItemExposureListener(yourRecyclerView, object : RVItemExposureListener.IOnExposureListener {
    override fun onExposure(position: Int) {
        // 滑动过程中出现的条目
        Log.d("exposure-curPosition:", position.toString())
    }

    override fun onUpload(exposureList: List<Int>?): Boolean {
        Log.d("exposure-positionList", exposureList.toString())
        // 上报成功后返回true
        return true
    }

})
```

---
完结，撒花🎉

