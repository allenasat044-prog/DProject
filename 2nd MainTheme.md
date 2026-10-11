You are given a task to integrate an existing React component in the codebase

The codebase should support:
- shadcn project structure  
- Tailwind CSS
- Typescript

If it doesn't, provide instructions on how to setup project via shadcn CLI, install Tailwind or Typescript.

Determine the default path for components and styles. 
If default path for components is not /components/ui, provide instructions on why it's important to create this folder
Copy-paste this component to /components/ui folder:
```tsx
sunset-skyline-hero.tsx
"use client"

import { useEffect, useRef, useState } from "react"

// ─────────────────────────────────────────────────────────────
// TOKYO SKYLINE HERO — locked scroll-scrub video hero
// Same technique as the metro/"City Opens" hero: body pinned with
// position:fixed while active, wheel/touch drive video.currentTime
// forward and backward, nothing else moves. Past the point where
// the source video ends, scroll keeps driving two extra layers —
// a composing Japanese brand mark and a transparent skyline
// cutout placed above it — before the page unlocks and continues.
// No dependencies, system fonts only.
// ─────────────────────────────────────────────────────────────

export interface NavItem {
  label: string
  href: string
}

export interface TokyoSkylineHeroProps {
  videoSrc?: string
  /** Transparent-background PNG of just the skyline silhouette, cropped identically to the video's last frame. */
  skylineSrc?: string
  title?: string
  scrollHint?: string
  /** Japanese brand mark that composes in once the video reaches its end. */
  brandMark?: string
  navItems?: NavItem[]
  signature?: { name: string; url: string } | false
  /** Total input distance (px) needed to scrub the full video. Tune to taste. */
  scrubDistance?: number
  className?: string
  style?: React.CSSProperties
}

const DEFAULT_VIDEO = "https://cdn.21st.dev/assets/mirror/23/234bc821170e75a6b8d2e42a952078f858eb054a51fab2a5baa928a10fdc245d.mp4"
const DEFAULT_SKYLINE = "https://cdn.21st.dev/assets/mirror/97/97fe4402ea1d5d05c2b82befe6dbf73da1d6669b995e6d5ae9f2ebea68b1107a.png"
const DEFAULT_SIGNATURE = { name: "guglielmogiannattasio.exe", url: "https://www.guglielmogiannattasio.it" }
const DEFAULT_NAV: NavItem[] = [
  { label: "ホーム", href: "#home" },
  { label: "作品", href: "#work" },
  { label: "予約", href: "#contact" },
]
const SANS = "-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif"

const COL_BG = "#1a0f0a"
const COL_TEXT = "#fff4e8"

// Progress breakpoints — tuned so the window disappears early (this
// footage clears it inside the first quarter) and the last ~20% of
// scroll is spent on the reveal rather than on the video itself.
const VIDEO_END = 0.78
const TEXT_START = 0.8
const TEXT_END = 0.93
const SKYLINE_START = 0.85
const SKYLINE_END = 0.96

function clamp(v: number, min: number, max: number) {
  return Math.min(max, Math.max(min, v))
}

export default function TokyoSkylineHero({
  videoSrc = DEFAULT_VIDEO,
  skylineSrc = DEFAULT_SKYLINE,
  title = "陽は、まだ沈んでいない。",
  scrollHint = "SCROLL",
  brandMark = "東京の窓",
  navItems = DEFAULT_NAV,
  signature = DEFAULT_SIGNATURE,
  scrubDistance = 3200,
  className,
  style,
}: TokyoSkylineHeroProps) {
  const sectionRef = useRef<HTMLDivElement>(null)
  const videoRef = useRef<HTMLVideoElement>(null)
  const titleRef = useRef<HTMLDivElement>(null)
  const hintRef = useRef<HTMLDivElement>(null)
  const brandRef = useRef<HTMLDivElement>(null)
  const skylineRef = useRef<HTMLImageElement>(null)
  const progressBarRef = useRef<HTMLDivElement>(null)
  const [ready, setReady] = useState(false)

  useEffect(() => {
    const video = videoRef.current
    const section = sectionRef.current
    if (!video || !section) return

    const reduceMotion =
      typeof window !== "undefined" &&
      window.matchMedia?.("(prefers-reduced-motion: reduce)").matches

    let duration = 0
    let rafId = 0
    let targetProgress = 0
    let currentProgress = 0
    let hasStartedScrolling = false
    let isSeeking = false
    let pendingTime: number | null = null
    let locked = false
    let lockedScrollY = 0
    let touchStartY = 0

    const onLoadedData = () => {
      duration = video.duration || 0
      setReady(true)
      if (reduceMotion) {
        video.currentTime = duration
      }
    }
    video.addEventListener("loadeddata", onLoadedData)

    const onSeeked = () => {
      isSeeking = false
      if (pendingTime !== null) {
        const t = pendingTime
        pendingTime = null
        isSeeking = true
        video.currentTime = t
      }
    }
    video.addEventListener("seeked", onSeeked)

    function seekTo(t: number) {
      if (isSeeking) {
        pendingTime = t
        return
      }
      isSeeking = true
      video!.currentTime = t
    }

    function engageLock() {
      if (locked || typeof document === "undefined") return
      locked = true
      lockedScrollY = window.scrollY
      const b = document.body.style
      b.position = "fixed"
      b.top = `-${lockedScrollY}px`
      b.left = "0"
      b.right = "0"
      b.width = "100%"
    }

    function releaseLock() {
      if (!locked || typeof document === "undefined") return
      locked = false
      const y = lockedScrollY
      const b = document.body.style
      b.position = ""
      b.top = ""
      b.left = ""
      b.right = ""
      b.width = ""
      window.scrollTo(0, y)
    }

    engageLock()

    function addDelta(deltaY: number) {
      const next = clamp(targetProgress + deltaY / scrubDistance, 0, 1)
      targetProgress = next
      if (targetProgress > 0.001) hasStartedScrolling = true
      return true
    }

    const onWheel = (e: WheelEvent) => {
      addDelta(e.deltaY)
      e.preventDefault()
    }

    const onTouchStart = (e: TouchEvent) => {
      touchStartY = e.touches[0]?.clientY ?? 0
    }
    const onTouchMove = (e: TouchEvent) => {
      const y = e.touches[0]?.clientY ?? touchStartY
      const deltaY = touchStartY - y
      touchStartY = y
      addDelta(deltaY)
      e.preventDefault()
    }

    window.addEventListener("wheel", onWheel, { passive: false })
    window.addEventListener("touchstart", onTouchStart, { passive: true })
    window.addEventListener("touchmove", onTouchMove, { passive: false })

    function frame() {
      currentProgress += (targetProgress - currentProgress) * 0.18

      if (duration > 0) {
        const videoT = clamp(currentProgress / VIDEO_END, 0, 1)
        seekTo(videoT * duration)
      }

      if (titleRef.current) {
        const t = 1 - clamp(currentProgress / 0.3, 0, 1)
        titleRef.current.style.opacity = String(t)
        titleRef.current.style.transform = `translateY(${(1 - t) * -24}px) scale(${0.96 + t * 0.04})`
        titleRef.current.style.filter = `blur(${(1 - t) * 10}px)`
      }
      if (hintRef.current) {
        hintRef.current.style.opacity = hasStartedScrolling ? "0" : "1"
      }
      if (brandRef.current) {
        const t = clamp((currentProgress - TEXT_START) / (TEXT_END - TEXT_START), 0, 1)
        brandRef.current.style.opacity = String(t)
        brandRef.current.style.transform = `translateY(${(1 - t) * 16}px) scale(${0.98 + t * 0.02})`
        brandRef.current.style.filter = `blur(${(1 - t) * 6}px)`
        brandRef.current.style.letterSpacing = `${(1 - t) * 0.25}em`
      }
      if (skylineRef.current) {
        const t = clamp((currentProgress - SKYLINE_START) / (SKYLINE_END - SKYLINE_START), 0, 1)
        skylineRef.current.style.opacity = String(t)
      }
      if (progressBarRef.current) {
        progressBarRef.current.style.transform = `scaleX(${currentProgress})`
      }

      rafId = requestAnimationFrame(frame)
    }

    if (!reduceMotion) {
      rafId = requestAnimationFrame(frame)
    }

    return () => {
      video.removeEventListener("loadeddata", onLoadedData)
      video.removeEventListener("seeked", onSeeked)
      window.removeEventListener("wheel", onWheel)
      window.removeEventListener("touchstart", onTouchStart)
      window.removeEventListener("touchmove", onTouchMove)
      cancelAnimationFrame(rafId)
      releaseLock()
    }
  }, [scrubDistance])

  return (
    <div
      ref={sectionRef}
      className={className}
      style={{
        position: "relative",
        height: "100dvh",
        width: "100%",
        overflow: "hidden",
        background: COL_BG,
        ...style,
      }}
    >
      <video
        ref={videoRef}
        src={videoSrc}
        muted
        playsInline
        preload="auto"
        style={{
          position: "absolute",
          inset: 0,
          width: "100%",
          height: "100%",
          objectFit: "cover",
          opacity: ready ? 1 : 0,
          transition: "opacity 0.6s ease",
        }}
      />

      {/* Japanese brand mark — sits above the video but below the skyline cutout, so buildings appear to sit in front of it. */}
      <div
        ref={brandRef}
        style={{
          position: "absolute",
          inset: 0,
          display: "flex",
          alignItems: "center",
          justifyContent: "center",
          padding: "0 8% 20vh",
                    textAlign: "center",
          opacity: 0,
          pointerEvents: "none",
          zIndex: 1,
        }}
      >
                <style>{`
         @media (max-width: 640px) {
  .tsh-brand-mark {
    writing-mode: vertical-rl;
    text-orientation: upright;
    font-size: clamp(64px, 22vw, 140px) !important;
    line-height: 1.15 !important;
    letter-spacing: 0.05em !important;
    transform: translate(-18%, -12%);
  }
}
        `}</style>
        <span
          className="tsh-brand-mark"
          style={{
            fontFamily: SANS,
            fontWeight: 800,
            fontSize: "clamp(56px, 15vw, 240px)",
            lineHeight: 1.05,
            letterSpacing: "-0.01em",
            color: COL_TEXT,
            textShadow: "0 8px 60px rgba(0,0,0,0.45)",
          }}
        >
          {brandMark}
        </span>
      </div>

      {/* Skyline cutout — transparent everywhere but the buildings, pixel-aligned with the video beneath it. */}
      <img
        ref={skylineRef}
        src={skylineSrc}
        alt=""
        aria-hidden="true"
        style={{
          position: "absolute",
          inset: 0,
          width: "100%",
          height: "100%",
          objectFit: "cover",
          opacity: 0,
          pointerEvents: "none",
          zIndex: 2,
        }}
      />

      <div
        style={{
          position: "absolute",
          inset: 0,
          background: "linear-gradient(180deg, rgba(26,15,10,0.35), rgba(26,15,10,0) 30%, rgba(26,15,10,0.15) 70%, rgba(26,15,10,0.55))",
          pointerEvents: "none",
          zIndex: 3,
        }}
      />

      <div
        ref={titleRef}
        style={{
          position: "absolute",
          inset: 0,
          display: "flex",
          alignItems: "center",
          justifyContent: "center",
          padding: "0 6%",
          textAlign: "center",
          pointerEvents: "none",
          zIndex: 3,
        }}
      >
        <span
          style={{
            fontFamily: SANS,
            fontWeight: 800,
            fontSize: "clamp(24px, 5.4vw, 64px)",
            lineHeight: 1.25,
            letterSpacing: "-0.01em",
            color: COL_TEXT,
            textShadow: "0 4px 30px rgba(0,0,0,0.5)",
          }}
        >
          {title}
        </span>
      </div>

      {/* Liquid navbar — always visible, independent of scroll progress. */}
      <nav
        style={{
          position: "absolute",
          top: "clamp(14px, 3vh, 28px)",
          left: "50%",
          transform: "translateX(-50%)",
          display: "flex",
          alignItems: "center",
          gap: "clamp(14px, 2.5vw, 28px)",
          padding: "clamp(8px, 1.4vw, 12px) clamp(18px, 3vw, 28px)",
          borderRadius: 999,
          background: "rgba(255,244,232,0.08)",
          backdropFilter: "blur(16px) saturate(160%)",
          WebkitBackdropFilter: "blur(16px) saturate(160%)",
          border: "1px solid rgba(255,244,232,0.14)",
          zIndex: 4,
        }}
      >
        {navItems.map((item) => (
          <a
            key={item.href}
            href={item.href}
            style={{
              fontFamily: SANS,
              fontWeight: 600,
              fontSize: "clamp(12px, 1.4vw, 14px)",
              color: "rgba(255,244,232,0.85)",
              textDecoration: "none",
              whiteSpace: "nowrap",
              transition: "color 0.2s ease",
            }}
            onMouseEnter={(e: React.MouseEvent<HTMLAnchorElement>) => {
              e.currentTarget.style.color = COL_TEXT
            }}
            onMouseLeave={(e: React.MouseEvent<HTMLAnchorElement>) => {
              e.currentTarget.style.color = "rgba(255,244,232,0.85)"
            }}
          >
            {item.label}
          </a>
        ))}
      </nav>

      <div
        ref={hintRef}
        style={{
          position: "absolute",
          left: "50%",
          bottom: "clamp(20px, 6vh, 48px)",
          transform: "translateX(-50%)",
          display: "flex",
          flexDirection: "column",
          alignItems: "center",
          gap: 8,
          color: "rgba(255,244,232,0.75)",
          fontFamily: SANS,
          fontSize: "clamp(10px, 1.4vw, 12px)",
          fontWeight: 600,
          letterSpacing: "0.3em",
          transition: "opacity 0.4s ease",
          pointerEvents: "none",
          zIndex: 4,
        }}
      >
        <span>{scrollHint}</span>
        <svg width="14" height="18" viewBox="0 0 14 18" style={{ animation: "tokyo-hero-bounce 1.6s ease-in-out infinite" }}>
          <style>{`
            @keyframes tokyo-hero-bounce {
              0%, 100% { transform: translateY(0); opacity: 0.5; }
              50% { transform: translateY(5px); opacity: 1; }
            }
          `}</style>
          <path d="M7 1 L7 17 M2 12 L7 17 L12 12" stroke="currentColor" strokeWidth="1.5" fill="none" strokeLinecap="round" strokeLinejoin="round" />
        </svg>
      </div>

      <div
        style={{
          position: "absolute",
          left: 0,
          right: 0,
          bottom: 0,
          height: 2,
          background: "rgba(255,244,232,0.12)",
          zIndex: 4,
        }}
      >
        <div
          ref={progressBarRef}
          style={{
            height: "100%",
            width: "100%",
            background: "linear-gradient(90deg, rgba(255,244,232,0.5), rgba(255,244,232,0.95))",
            transform: "scaleX(0)",
            transformOrigin: "left center",
          }}
        />
      </div>

      {signature && (
        <span
          style={{
            position: "absolute",
            right: "clamp(12px, 2.5vw, 24px)",
            bottom: "clamp(10px, 2vw, 18px)",
            fontFamily: SANS,
            fontWeight: 500,
            fontSize: "clamp(11px, 1.4vw, 13px)",
            letterSpacing: "0.01em",
            color: "rgba(255,244,232,0.6)",
            zIndex: 4,
          }}
        >
          by{" "}
          <a
            href={signature.url}
            target="_blank"
            rel="noopener noreferrer"
            style={{
              color: "rgba(255,244,232,0.6)",
              textDecoration: "none",
              transition: "color 0.2s ease",
            }}
            onMouseEnter={(e: React.MouseEvent<HTMLAnchorElement>) => {
              e.currentTarget.style.color = COL_TEXT
            }}
            onMouseLeave={(e: React.MouseEvent<HTMLAnchorElement>) => {
              e.currentTarget.style.color = "rgba(255,244,232,0.6)"
            }}
          >
            {signature.name}
          </a>
        </span>
      )}
    </div>
  )
}


demo.tsx
import TokyoSkylineHero from "@/components/ui/sunset-skyline-hero"

export default function Demo() {
  return <TokyoSkylineHero />
}
```

Implementation Guidelines
 1. Analyze the component structure and identify all required dependencies
 2. Review the component's argumens and state
 3. Identify any required context providers or hooks and install them
 4. Questions to Ask
 - What data/props will be passed to this component?
 - Are there any specific state management requirements?
 - Are there any required assets (images, icons, etc.)?
 - What is the expected responsive behavior?
 - What is the best place to use this component in the app?

Steps to integrate
 0. Copy paste all the code above in the correct directories
 1. Install external dependencies
 2. Fill image assets with Unsplash stock images you know exist
 3. Use lucide-react icons for svgs or logos if component requires them
