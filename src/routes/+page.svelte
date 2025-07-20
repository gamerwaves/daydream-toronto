<script lang="ts">
	import { onMount } from "svelte";

	function createSmoothPath(points: Array<{ x: number; y: number }>) {
		if (points.length < 2) return "";
		
		// Create smooth curves that flow horizontally through points
		const tension = 1.2; // Increased control point distance for smoother curves
		
		// Configurable angles for each point (in degrees)
		// 0 = horizontal, positive = upward slope, negative = downward slope
		const pointAngles = [0, 0, -10, -10, -10, 0, 0]; // Point 0: center, Point 0.5: left curve, Points 1-4: cards, Point 5: final card
		
		// Generate control points for smooth curves
		const controlPoints = [];
		
		for (let i = 0; i < points.length; i++) {
			let cp1, cp2;
			const angleRadians = (pointAngles[i] || 0) * Math.PI / 180; // Convert to radians
			
			if (i === 0) {
				// First point - start flowing at specified angle
				const next = points[i + 1];
				const distance = Math.sqrt(Math.pow(next.x - points[i].x, 2) + Math.pow(next.y - points[i].y, 2));
				
				// Start at specified angle - extend further
				const controlDistance = distance * tension * 0.8;
				
				cp1 = { x: points[i].x, y: points[i].y };
				cp2 = { 
					x: points[i].x + Math.cos(angleRadians) * controlDistance, 
					y: points[i].y + Math.sin(angleRadians) * controlDistance
				};
			} else if (i === points.length - 1) {
				// Last point - simple straight-down vertical approach
				const prev = points[i - 1];
				const distance = Math.sqrt(Math.pow(points[i].x - prev.x, 2) + Math.pow(points[i].y - prev.y, 2));
				
				// Create a simple curve that transitions smoothly to vertical
				const controlDistance = distance * tension * 0.4; // Reduced for simpler curve
				
				// First control point - minimal horizontal extension from previous point
				cp1 = { 
					x: prev.x + (points[i].x - prev.x) * 0.7, // Move most of the way horizontally
					y: prev.y + (points[i].y - prev.y) * 0.3   // Move only slightly vertically
				};
				
				// Second control point - directly above target for straight-down approach
				cp2 = { 
					x: points[i].x, // Directly above the target point
					y: points[i].y - controlDistance * 0.5 // Close approach from above
				};
			} else {
				// Middle points - flow at specified angle through the point
				const prev = points[i - 1];
				const next = points[i + 1];
				
				// Calculate distances for proportional control
				const prevDistance = Math.sqrt(Math.pow(points[i].x - prev.x, 2) + Math.pow(points[i].y - prev.y, 2));
				const nextDistance = Math.sqrt(Math.pow(next.x - points[i].x, 2) + Math.pow(next.y - points[i].y, 2));
				
				// Determine if we're going left-to-right or right-to-left based on position in sequence
				const isEven = i % 2 === 0;
				const flowDirection = isEven ? 1 : -1; // Even indices flow right, odd flow left
				
				// Control distances based on adjacent point distances - extend much further
				const leftControlDistance = prevDistance * tension * 0.7;
				const rightControlDistance = nextDistance * tension * 0.7;
				
				if (flowDirection > 0) {
					// Flowing left to right at specified angle
					cp1 = { 
						x: points[i].x - Math.cos(angleRadians) * leftControlDistance, 
						y: points[i].y - Math.sin(angleRadians) * leftControlDistance
					};
					cp2 = { 
						x: points[i].x + Math.cos(angleRadians) * rightControlDistance, 
						y: points[i].y + Math.sin(angleRadians) * rightControlDistance
					};
				} else {
					// Flowing right to left at specified angle (flip direction)
					cp1 = { 
						x: points[i].x + Math.cos(angleRadians) * leftControlDistance, 
						y: points[i].y + Math.sin(angleRadians) * leftControlDistance
					};
					cp2 = { 
						x: points[i].x - Math.cos(angleRadians) * rightControlDistance, 
						y: points[i].y - Math.sin(angleRadians) * rightControlDistance
					};
				}
			}
			
			controlPoints.push({ cp1, cp2 });
		}
		
		// Build the smooth path
		let path = `M ${points[0].x} ${points[0].y}`;
		
		for (let i = 1; i < points.length; i++) {
			const prevControls = controlPoints[i - 1];
			const currControls = controlPoints[i];
			
			path += ` C ${prevControls.cp2.x} ${prevControls.cp2.y}, ${currControls.cp1.x} ${currControls.cp1.y}, ${points[i].x} ${points[i].y}`;
		}
		
		return path;
	}

	function getPointAlongPath(points: Array<{ x: number; y: number }>, percentage: number) {
		if (points.length < 2) return { x: 0, y: 0, angle: 0 };
		
		// Generate the same control points as the path
		const tension = 1.2;
		const pointAngles = [0, 0, -10, -10, -10, 0, 0];
		const controlPoints = [];
		
		for (let i = 0; i < points.length; i++) {
			let cp1, cp2;
			const angleRadians = (pointAngles[i] || 0) * Math.PI / 180;
			
			if (i === 0) {
				const next = points[i + 1];
				const distance = Math.sqrt(Math.pow(next.x - points[i].x, 2) + Math.pow(next.y - points[i].y, 2));
				const controlDistance = distance * tension * 0.8;
				
				cp1 = { x: points[i].x, y: points[i].y };
				cp2 = { 
					x: points[i].x + Math.cos(angleRadians) * controlDistance, 
					y: points[i].y + Math.sin(angleRadians) * controlDistance
				};
			} else if (i === points.length - 1) {
				const prev = points[i - 1];
				const distance = Math.sqrt(Math.pow(points[i].x - prev.x, 2) + Math.pow(points[i].y - prev.y, 2));
				const controlDistance = distance * tension * 0.4;
				
				cp1 = { 
					x: prev.x + (points[i].x - prev.x) * 0.7, 
					y: prev.y + (points[i].y - prev.y) * 0.3
				};
				
				cp2 = { 
					x: points[i].x, 
					y: points[i].y - controlDistance * 0.5
				};
			} else {
				const prev = points[i - 1];
				const next = points[i + 1];
				
				const prevDistance = Math.sqrt(Math.pow(points[i].x - prev.x, 2) + Math.pow(points[i].y - prev.y, 2));
				const nextDistance = Math.sqrt(Math.pow(next.x - points[i].x, 2) + Math.pow(next.y - points[i].y, 2));
				
				const isEven = i % 2 === 0;
				const flowDirection = isEven ? 1 : -1;
				
				const leftControlDistance = prevDistance * tension * 0.7;
				const rightControlDistance = nextDistance * tension * 0.7;
				
				if (flowDirection > 0) {
					cp1 = { 
						x: points[i].x - Math.cos(angleRadians) * leftControlDistance, 
						y: points[i].y - Math.sin(angleRadians) * leftControlDistance
					};
					cp2 = { 
						x: points[i].x + Math.cos(angleRadians) * rightControlDistance, 
						y: points[i].y + Math.sin(angleRadians) * rightControlDistance
					};
				} else {
					cp1 = { 
						x: points[i].x + Math.cos(angleRadians) * leftControlDistance, 
						y: points[i].y + Math.sin(angleRadians) * leftControlDistance
					};
					cp2 = { 
						x: points[i].x - Math.cos(angleRadians) * rightControlDistance, 
						y: points[i].y - Math.sin(angleRadians) * rightControlDistance
					};
				}
			}
			
			controlPoints.push({ cp1, cp2 });
		}
		
		// Calculate which segment and position within that segment
		const segmentCount = points.length - 1;
		const segmentPercentage = percentage * segmentCount;
		const segmentIndex = Math.floor(segmentPercentage);
		const t = segmentPercentage - segmentIndex;
		
		// Clamp to valid range
		const clampedIndex = Math.min(segmentIndex, segmentCount - 1);
		const clampedT = clampedIndex === segmentIndex ? t : 1;
		
		// Get the Bézier curve points for this segment
		const p0 = points[clampedIndex];
		const p1 = controlPoints[clampedIndex].cp2;
		const p2 = controlPoints[clampedIndex + 1].cp1;
		const p3 = points[clampedIndex + 1];
		
		// Calculate position on cubic Bézier curve
		const x = Math.pow(1 - clampedT, 3) * p0.x + 
				  3 * Math.pow(1 - clampedT, 2) * clampedT * p1.x + 
				  3 * (1 - clampedT) * Math.pow(clampedT, 2) * p2.x + 
				  Math.pow(clampedT, 3) * p3.x;
		
		const y = Math.pow(1 - clampedT, 3) * p0.y + 
				  3 * Math.pow(1 - clampedT, 2) * clampedT * p1.y + 
				  3 * (1 - clampedT) * Math.pow(clampedT, 2) * p2.y + 
				  Math.pow(clampedT, 3) * p3.y;
		
		// Calculate tangent for rotation
		const dx = 3 * Math.pow(1 - clampedT, 2) * (p1.x - p0.x) + 
				   6 * (1 - clampedT) * clampedT * (p2.x - p1.x) + 
				   3 * Math.pow(clampedT, 2) * (p3.x - p2.x);
		
		const dy = 3 * Math.pow(1 - clampedT, 2) * (p1.y - p0.y) + 
				   6 * (1 - clampedT) * clampedT * (p2.y - p1.y) + 
				   3 * Math.pow(clampedT, 2) * (p3.y - p2.y);
		
		let angle = Math.atan2(dy, dx) * 180 / Math.PI;
		
		// Keep the plane oriented to follow the curve naturally
		// Allow flipping but ensure it's always pointing in the direction of travel
		
		return { x, y, angle };
	}



	let currentAirplaneProgress = 0;
	let animationFrameId: number | null = null;
	let isFlipped = false;

	function handleFormSubmit(event: Event) {
		event.preventDefault();
		const form = event.target as HTMLFormElement;
		const emailInput = form.querySelector('input[name="email"]') as HTMLInputElement;
		const email = emailInput.value;
		
		// Find the hidden form and submit it
		const hiddenForm = document.getElementById('hidden-signup-form') as HTMLFormElement;
		const hiddenEmailInput = hiddenForm.querySelector('input[name="email"]') as HTMLInputElement;
		hiddenEmailInput.value = email;
		hiddenForm.submit();
		
		// Open the Hackclub registration form with email parameter for Toronto event
		window.open(`https://forms.hackclub.com/daydream-Toronto?email=${encodeURIComponent(email)}`, '_blank');
		
		// Clear the email input
		emailInput.value = '';
	}

	function updateAirplanePosition() {
		const container = document.getElementById("islands-container");
		const airplane = document.getElementById("paper-airplane");
		
		if (!container || !airplane) return;
		
		const containerRect = container.getBoundingClientRect();
		const windowHeight = window.innerHeight;
		
		// Calculate scroll progress based on container position
		// Start animation earlier and extend it longer for full path completion
		const scrollStart = containerRect.top + window.scrollY - windowHeight * 0.7;
		const scrollEnd = containerRect.bottom + window.scrollY - windowHeight;
		const currentScroll = window.scrollY;
		
		// Calculate progress (0 to 1)
		let progress = (currentScroll - scrollStart) / (scrollEnd - scrollStart);
		progress = Math.max(0, Math.min(1, progress));
		
		// Apply ease-in-out curve for smoother animation
		const easingStrength = 1.0; // 0 = linear, 1 = full ease-in-out
		const easedProgress = progress * progress * (3 - 2 * progress);
		progress = progress * (1 - easingStrength) + easedProgress * easingStrength;
		
		// Smooth the movement with interpolation and max speed
		const maxSpeed = 0.001; // Maximum progress change per frame
		const targetDifference = progress - currentAirplaneProgress;
		const speedLimitedDifference = Math.sign(targetDifference) * Math.min(Math.abs(targetDifference), maxSpeed);
		const previousProgress = currentAirplaneProgress;
		currentAirplaneProgress += speedLimitedDifference;
		
		// Determine movement direction
		const movingForward = currentAirplaneProgress > previousProgress;
		

		
		// Get points for path calculation
		const points: Array<{ x: number; y: number }> = [];
		const pointIds = ["0", "0.5", "1", "2", "3", "4", "5"];
		pointIds.forEach(id => {
			const element = document.querySelector(`[data-point="${id}"]`);
			if (element) {
				const rect = element.getBoundingClientRect();
				points.push({
					x: rect.left + rect.width / 2 - containerRect.left,
					y: rect.top + rect.height / 2 - containerRect.top
				});
			}
		});
		
		if (points.length > 0) {
			const airplanePos = getPointAlongPath(points, currentAirplaneProgress);
			airplane.style.left = `${airplanePos.x}px`;
			airplane.style.top = `${airplanePos.y}px`;
			
			// Check if rotation angle is greater than 90 degrees (plane is upside down)
			// Normalize angle to -180 to 180 range
			let normalizedAngle = airplanePos.angle;
			while (normalizedAngle > 180) normalizedAngle -= 360;
			while (normalizedAngle < -180) normalizedAngle += 360;
			
			// Flip plane if angle is outside -90 to 90 degree range (keeps plane right side up)
			isFlipped = Math.abs(normalizedAngle) > 90;
			
			// Apply vertical flip if needed
			const verticalFlip = isFlipped ? ' scaleY(-1)' : '';
			airplane.style.transform = `translate(-50%, calc(-50% - 0.5rem)) rotate(${airplanePos.angle}deg)${verticalFlip}`;
		}
		
		// Continue animation if still moving
		if (Math.abs(progress - currentAirplaneProgress) > 0.001) {
			animationFrameId = requestAnimationFrame(updateAirplanePosition);
		}
	}

	function updatePath() {
		const container = document.getElementById("islands-container");
		const pathElement = document.getElementById("dotted-path");
		
		if (!container || !pathElement) return;
		
		const containerRect = container.getBoundingClientRect();
		const points: Array<{ x: number; y: number }> = [];
		
		// Get points in order by data-point attribute
		const pointIds = ["0", "0.5", "1", "2", "3", "4", "5"];
		pointIds.forEach(id => {
			const element = document.querySelector(`[data-point="${id}"]`);
			if (element) {
				const rect = element.getBoundingClientRect();
				points.push({
					x: rect.left + rect.width / 2 - containerRect.left,
					y: rect.top + rect.height / 2 - containerRect.top
				});
			}
		});
		
		const pathData = createSmoothPath(points);
		pathElement.setAttribute("d", pathData);
		
		// Update airplane position based on scroll
		updateAirplanePosition();
	}

	onMount(() => {
		// Initial path calculation
		setTimeout(updatePath, 100);
		
		// Update on scroll and resize
		window.addEventListener("scroll", updateAirplanePosition);
		window.addEventListener("resize", updatePath);
		
		// Cleanup
		return () => {
			window.removeEventListener("scroll", updateAirplanePosition);
			window.removeEventListener("resize", updatePath);
			if (animationFrameId) {
				cancelAnimationFrame(animationFrameId);
			}
		};
	});
</script>

<style>
	:global(body) {
		background-color: #CCF4FD;
		max-width: 100vw;
		overflow-x: hidden;	
	}
	:global(html) {
		overflow-x: hidden;
	}
</style>

<svelte:head>
	<title>Daydream Toronto - Game Jam for High Schoolers</title>
</svelte:head>

<div class="absolute top-0 left-0 w-full h-full bg-[url('brushstroking.png')] bg-size-[100vw_100vh] bg-repeat mix-blend-overlay opacity-60 pointer-events-none"></div>

<div class="flex flex-col items-center justify-center h-screen text-center bg-gradient-to-b from-[#CCF4FD] to-[#B8D9F8] bg-blend-overlay relative">
	<div class="absolute top-0 left-0 w-full h-full bg-[url('brushstroking.png')] bg-size-[100vw_100vh] bg-repeat mix-blend-overlay opacity-30 pointer-events-none"></div>

	<div class="absolute top-0 left-0 w-full h-full bg-[url(/buildings-back.png)] bg-no-repeat bg-contain pointer-events-none lg:-translate-y-15"></div>
	<div class="absolute top-0 left-0 w-full h-full bg-[url(/buildings-front.png)] bg-no-repeat bg-contain pointer-events-none lg:-translate-y-15"></div>
	<!-- brush texture clipped to buildings -->
	<div class="absolute top-0 left-0 w-full h-full bg-[url('brushstroking.png')] bg-size-[100vw_100vh] bg-repeat pointer-events-none opacity-100 -translate-y-15 bg-center mix-blend-overlay" style="mask-image: url('/buildings-top.png'); mask-size: contain; mask-repeat: no-repeat; mask-position: center top; -webkit-mask-image: url('/buildings-top.png'); -webkit-mask-size: contain; -webkit-mask-repeat: no-repeat; -webkit-mask-position: center top;"></div>
	<div class="inline-block relative">
		<div class="h-12"></div> 
		<!-- space for the ship -->
		<h2
		class="text-xl font-serif bg-gradient-to-b from-[#487DAB] to-[#3F709A] bg-clip-text text-transparent absolute left-1/2 max-sm:translate-y-4 max-sm:mb-0 max-md:-mb-8 md:left-[calc(50%+4rem)] -translate-x-1/2 bottom-8 italic w-max md:text-lg max-sm:text-lg"
		>
			
		</h2>
		<img src="daydream.png" alt="Daydream" class="h-40 mb-6 w-auto object-contain max-w-full px-4" />
		<a href="https://hackclub.com" class="absolute top-0 -right-6 max-sm:right-0 max-sm:scale-80 animate-hover ![animation-delay:0.9s] ![--hover:-0.2rem]">
			<img src="flag-plane.png" alt="Hack Club" class="h-28">
		</a>
	</div>
	<div class="relative inline-block px-4">
		<h3
			class="text-3xl italic font-serif bg-gradient-to-b from-[#487DAB] to-[#3F709A] bg-clip-text text-transparent w-max max-sm:text-2xl mx-auto"
		>
			Build amazing games, win amazing prizes.
		</h3>
		<img
			src="underline.svg"
			alt=""
			class="absolute left-1/2 -translate-x-1/2 -mt-1 h-auto scale-115"
		/>
		<h4
			class="text-2xl opacity-90 mt-2 font-serif bg-gradient-to-b from-[#487DAB] to-[#3F709A] bg-clip-text text-transparent max-sm:text-xl"
		>
			Toronto - September 27th & 28th, 2025
		</h4>
	</div>
	
	<div class="mt-8 flex flex-col items-center gap-3 z-10 max-md:scale-90">
		<a
			href="https://forms.hackclub.com/daydream-Toronto"
			target="_blank"
			class="inline-block px-8 py-4 bg-pink border-b-4 border-b-pink-dark text-white rounded-full hover:shadow-[0_4px_0_0_theme(colors.pink.dark)] hover:-translate-y-[4px] active:border-transparent active:shadow-none active:translate-y-0 transition-all duration-100 font-sans text-xl cursor-pointer relative overflow-hidden"
		>
			Join Now
			<img
				src="button-clouds.svg" 
				alt="" 
				class="absolute bottom-0 left-1/2 -translate-x-1/2 w-auto object-contain pointer-events-none"
			>
		</a>
	</div>

	<!-- <img src="hot-air-balloon.png" alt="" class="absolute w-1/8 right-32 bottom-40 z-20"> -->
	<!-- <img src="hot-air-balloon.png" alt="" class="absolute w-1/12 left-36 bottom-81 z-20"> -->

	<img src="/clouds-top-middle-bg.svg" alt="" class="absolute left-5/12 -translate-x-1/2 w-7/12 -bottom-24">
	<div class="absolute left-5/12 -translate-x-1/2 w-7/12 -bottom-24 bg-[url('brushstroking.png')] bg-size-[100vw_100vh] bg-repeat mix-blend-overlay opacity-60 pointer-events-none h-full" style="mask-image: url('/clouds-top-middle-bg.svg'); mask-size: contain; mask-repeat: no-repeat; mask-position: center; -webkit-mask-image: url('/clouds-top-middle-bg.svg'); -webkit-mask-size: contain; -webkit-mask-repeat: no-repeat; -webkit-mask-position: center;"></div>
	
	<img src="/clouds-top-right-bg.svg" alt="" class="absolute right-0 w-1/2 -bottom-12 translate-y-1/2">
	<div class="absolute right-0 w-1/2 -bottom-12 translate-y-1/2 bg-[url('brushstroking.png')] bg-size-[100vw_100vh] bg-repeat mix-blend-overlay opacity-60 pointer-events-none h-full" style="mask-image: url('/clouds-top-right-bg.svg'); mask-size: contain; mask-repeat: no-repeat; mask-position: center; -webkit-mask-image: url('/clouds-top-right-bg.svg'); -webkit-mask-size: contain; -webkit-mask-repeat: no-repeat; -webkit-mask-position: center;"></div>
	
	<img src="/clouds-top-left-bg.svg" alt="" class="absolute left-0 w-3/12 -bottom-12  translate-y-1/2">
	<div class="absolute left-0 w-3/12 -bottom-12 translate-y-1/2 bg-[url('brushstroking.png')] bg-size-[100vw_100vh] bg-repeat mix-blend-overlay opacity-60 pointer-events-none h-full" style="mask-image: url('/clouds-top-left-bg.svg'); mask-size: contain; mask-repeat: no-repeat; mask-position: center; -webkit-mask-image: url('/clouds-top-left-bg.svg'); -webkit-mask-size: contain; -webkit-mask-repeat: no-repeat; -webkit-mask-position: center;"></div>
	
	<img src="/clouds-top-middle.png" alt="" class="absolute left-5/12 -translate-x-1/2 w-7/12 -bottom-24">
	<img src="/clouds-top-right.png" alt="" class="absolute right-0 w-1/2 -bottom-12 translate-y-1/2">
	<img src="/clouds-top-left.png" alt="" class="absolute left-0 w-3/12 -bottom-12  translate-y-1/2">
	
	<!-- Cloudy Background -->
	<div class="absolute bottom-0 left-1/2 -translate-x-1/2 w-full h-[80vh] bg-[url(/cloudy-bg.png)] opacity-30 bg-cover bg-no-repeat bg-top pointer-events-none -z-10"></div>
</div>

<div class="w-full relative flex items-start justify-center min-h-screen">
	<!-- background -->
	<div class="absolute top-0 left-0 w-full h-full -z-50 bg-[#FCEFC5]"></div>
	<div class="absolute top-0 left-0 w-full h-full bg-[url('brushstroking.png')] bg-size-[100vw_100vh] bg-repeat mix-blend-overlay opacity-30 pointer-events-none -z-40"></div>
	
	<div class="relative max-w-4xl mx-auto h-full flex items-start pt-24 max-sm:pt-40 px-8 max-sm:px-2">
		<!-- Add decorative elements -->
		<img src="/hot-air-balloon.png" alt="" class="absolute -top-10 -right-10 w-24 h-24 opacity-60 pointer-events-none z-10 animate-bounce ![animation-duration:3s]">
		<img src="/thought-bubbles.png" alt="" class="absolute top-20 -left-16 w-20 h-20 opacity-40 pointer-events-none z-10">
		
		<div class="relative z-20 px-20 pt-20 pb-32 rounded-lg mb-0 max-sm:px-18" style="background-image: url('/letter-top.png'), linear-gradient(to bottom, #FCEFC5 100px, transparent 100px), url('/letter-loop.png'); background-size: 100% auto, 100% auto, 100% auto; background-repeat: no-repeat, no-repeat, repeat-y; background-position: top, top, top; background-attachment: local, local, local;">
			<div class="absolute bottom-0 left-0 w-full h-24 z-10 pointer-events-none bg-[url('/clouds-loop.png')] bg-repeat-x bg-bottom bg-contain"></div>
			
			<!-- Add some decorative game-themed icons -->
			<div class="absolute top-6 right-6 opacity-20 pointer-events-none">
				<div class="text-6xl">🎮</div>
			</div>
			<div class="absolute top-16 left-6 opacity-15 pointer-events-none">
				<div class="text-4xl">⭐</div>
			</div>
			
			<h2 class="text-5xl font-serif italic text-[#8B4513] mb-10 relative">
				What's Daydream Toronto?
				<img src="/underline.svg" alt="" class="absolute left-0 -bottom-3 w-64 h-auto opacity-70">
			</h2>
			
			<div class="text-[#8B4513] font-serif text-xl leading-relaxed space-y-8">
				<div class="relative">
					<p class="relative z-10">
						Daydream Toronto is a <em class="italic font-bold bg-gradient-to-r from-[#E472AB] to-[#F2993E] bg-clip-text text-transparent">game jam</em> for high schoolers happening in Toronto, 
						where you can build the most creative games you can think of! Whether it's a platformer that defies physics, 
						a puzzle game that makes people think differently, or that wild game idea you've always wanted to try - 
						Daydream is the place to make it happen.
					</p>
					<!-- Add subtle floating game elements -->
					<div class="absolute -right-8 top-0 opacity-30 pointer-events-none text-3xl animate-pulse ![animation-duration:2s]">🚀</div>
				</div>
				
				<div class="relative">
					<p>
						No matter your experience level, Daydream Toronto needs you and your creative ideas! We'll provide workshops, 
						mentors, and everything you need to bring your game concepts to life.
					</p>
					<div class="absolute -left-6 top-2 opacity-25 pointer-events-none text-2xl animate-pulse ![animation-duration:2.5s] ![animation-delay:0.5s]">💡</div>
				</div>
				
				<div class="relative text-center my-12">
					<p class="font-bold text-3xl bg-gradient-to-r from-[#8B4513] to-[#CD853F] bg-clip-text text-transparent relative inline-block">
						What's happening at Daydream Toronto?
						<img src="/underline.svg" alt="" class="absolute left-1/2 -translate-x-1/2 -bottom-2 w-full h-auto opacity-50">
					</p>
					<div class="absolute -top-2 -right-8 opacity-30 pointer-events-none text-2xl">🎯</div>
					<div class="absolute -top-2 -left-8 opacity-30 pointer-events-none text-2xl">🏆</div>
				</div>
				
				<p class="text-center text-lg">
					Daydream Toronto is a <span class="font-bold text-2xl bg-gradient-to-r from-[#E472AB] to-[#639DEB] bg-clip-text text-transparent">24-hour event</span> - Here's a rough schedule!
				</p>
				
				<div class="bg-gradient-to-br from-[#F5F5DC] to-[#FFF8DC] p-8 rounded-2xl border-3 border-[#D2B48C] text-base space-y-3 shadow-lg relative overflow-hidden">
					<!-- Add decorative background pattern -->
					<div class="absolute top-0 right-0 w-32 h-32 opacity-10 pointer-events-none">
						<div class="text-8xl">⏰</div>
					</div>
					<div class="absolute bottom-0 left-0 w-24 h-24 opacity-10 pointer-events-none">
						<div class="text-6xl">🎮</div>
					</div>
					
					<h4 class="font-bold text-xl text-[#8B4513] mb-4 text-center border-b-2 border-[#D2B48C] pb-2">Event Schedule</h4>
					
					<div class="grid gap-2 relative z-10">
						<div class="flex justify-between items-center bg-white/50 rounded-lg p-3 border border-[#DEB887]">
							<span class="font-bold flex items-center gap-2"><span class="text-lg">🚪</span> Doors open & Check-in</span>
							<span class="bg-[#E472AB] text-white px-3 py-1 rounded-full text-sm font-bold">Saturday 1:00 PM</span>
						</div>
						<div class="flex justify-between items-center bg-white/50 rounded-lg p-3 border border-[#DEB887]">
							<span class="font-bold flex items-center gap-2"><span class="text-lg">🎉</span> Opening ceremony</span>
							<span class="bg-[#639DEB] text-white px-3 py-1 rounded-full text-sm font-bold">2:00 PM</span>
						</div>
						<div class="flex justify-between items-center bg-white/50 rounded-lg p-3 border border-[#DEB887]">
							<span class="font-bold flex items-center gap-2"><span class="text-lg">⚡</span> Start building your game!</span>
							<span class="bg-[#F2993E] text-white px-3 py-1 rounded-full text-sm font-bold">2:30 PM</span>
						</div>
						<div class="flex justify-between items-center bg-white/50 rounded-lg p-3 border border-[#DEB887]">
							<span class="font-bold flex items-center gap-2"><span class="text-lg">📚</span> Workshop: Game Design Basics</span>
							<span class="bg-[#AB68E2] text-white px-3 py-1 rounded-full text-sm font-bold">4:00 PM</span>
						</div>
						<div class="flex justify-between items-center bg-white/50 rounded-lg p-3 border border-[#DEB887]">
							<span class="font-bold flex items-center gap-2"><span class="text-lg">🍽️</span> Dinner</span>
							<span class="bg-[#E472AB] text-white px-3 py-1 rounded-full text-sm font-bold">6:00 PM</span>
						</div>
						<div class="flex justify-between items-center bg-white/50 rounded-lg p-3 border border-[#DEB887]">
							<span class="font-bold flex items-center gap-2"><span class="text-lg">🎨</span> Workshop: Art & Sound for Games</span>
							<span class="bg-[#639DEB] text-white px-3 py-1 rounded-full text-sm font-bold">8:00 PM</span>
						</div>
						<div class="flex justify-between items-center bg-white/50 rounded-lg p-3 border border-[#DEB887]">
							<span class="font-bold flex items-center gap-2"><span class="text-lg">🌙</span> Late night coding & pizza</span>
							<span class="bg-[#F2993E] text-white px-3 py-1 rounded-full text-sm font-bold">11:00 PM</span>
						</div>
						<div class="flex justify-between items-center bg-white/50 rounded-lg p-3 border border-[#DEB887]">
							<span class="font-bold flex items-center gap-2"><span class="text-lg">🎊</span> Midnight surprise activity</span>
							<span class="bg-[#AB68E2] text-white px-3 py-1 rounded-full text-sm font-bold">12:00 AM</span>
						</div>
						<div class="flex justify-between items-center bg-white/50 rounded-lg p-3 border border-[#DEB887]">
							<span class="font-bold flex items-center gap-2"><span class="text-lg">🌅</span> Breakfast</span>
							<span class="bg-[#E472AB] text-white px-3 py-1 rounded-full text-sm font-bold">Sunday 8:00 AM</span>
						</div>
						<div class="flex justify-between items-center bg-white/50 rounded-lg p-3 border border-[#DEB887]">
							<span class="font-bold flex items-center gap-2"><span class="text-lg">✨</span> Final polish time</span>
							<span class="bg-[#639DEB] text-white px-3 py-1 rounded-full text-sm font-bold">10:00 AM</span>
						</div>
						<div class="flex justify-between items-center bg-white/50 rounded-lg p-3 border border-[#DEB887]">
							<span class="font-bold flex items-center gap-2"><span class="text-lg">🎤</span> Project demos & judging</span>
							<span class="bg-[#F2993E] text-white px-3 py-1 rounded-full text-sm font-bold">12:00 PM</span>
						</div>
						<div class="flex justify-between items-center bg-white/50 rounded-lg p-3 border border-[#DEB887]">
							<span class="font-bold flex items-center gap-2"><span class="text-lg">🏆</span> Closing ceremony & prizes</span>
							<span class="bg-[#AB68E2] text-white px-3 py-1 rounded-full text-sm font-bold">1:30 PM</span>
						</div>
					</div>
				</div>
				
				<div class="text-center relative">
					<div class="bg-gradient-to-r from-[#FAE3C9] to-[#F5E6D3] p-8 rounded-2xl border-2 border-[#AA8B83] shadow-xl relative overflow-hidden">
						<img src="button-clouds.svg" alt="" class="absolute bottom-0 left-1/2 -translate-x-1/2 w-full opacity-20 pointer-events-none">
						<div class="absolute top-2 left-2 opacity-20 pointer-events-none text-3xl">🎮</div>
						<div class="absolute top-2 right-2 opacity-20 pointer-events-none text-3xl">🚀</div>
						
						<p class="font-bold text-3xl mb-6 bg-gradient-to-r from-[#2C3E50] to-[#8B4513] bg-clip-text text-transparent relative">
							Ready for an amazing weekend?
						</p>
						<a 
							href="https://forms.hackclub.com/daydream-Toronto"
							class="inline-block px-8 py-4 bg-gradient-to-r from-[#E472AB] to-[#F2993E] text-white rounded-full hover:shadow-[0_8px_20px_rgba(228,114,171,0.4)] hover:-translate-y-1 active:translate-y-0 transition-all duration-200 font-sans text-xl font-bold cursor-pointer relative overflow-hidden transform hover:scale-105"
						>
							Sign up now! ✨
							<div class="absolute inset-0 bg-gradient-to-r from-white/20 to-transparent opacity-0 hover:opacity-100 transition-opacity duration-200"></div>
						</a>
					</div>
				</div>
			</div>
		</div>
	</div>

	<div class="w-full absolute z-30 max-h-64 bottom-0 translate-y-16 max-xl:translate-y-20 max-lg:translate-y-24 pointer-events-none">	
		<img src="/cloud-cover-1.png" alt="" class="w-full h-full object-contain">
		<div class="absolute top-1/2 left-1/2 w-1 h-1 -translate-x-1/2 -translate-y-1/2" data-point="0"></div>
		<div class="absolute top-1/2 left-1/4 w-1 h-1 -translate-x-1/2 -translate-y-1/2" data-point="0.5"></div>
	</div>
</div>

<!-- Location Section -->
<div class="w-full bg-[#FCEFC5] py-20 pt-96 relative">
	<div class="absolute top-0 left-0 w-full h-full bg-[url('brushstroking.png')] bg-size-[100vw_100vh] bg-repeat mix-blend-overlay opacity-30 pointer-events-none"></div>
	
	<div class="max-w-4xl mx-auto text-center px-8">
		<h2 class="text-5xl font-serif italic text-[#8B4513] mb-8">
			Where is Daydream Toronto?
		</h2>
		
		<div class="bg-white p-8 rounded-lg border-2 border-[#AA8B83] shadow-lg inline-block">
			<p class="text-2xl font-serif text-[#8B4513] mb-4">
				The event will be held at <span class="font-bold">[Venue Name]</span>
			</p>
			<p class="text-lg text-[#8B4513] mb-6">
				[Street Address]<br>
				Toronto, ON [Postal Code], Canada
			</p>
			<p class="text-base text-[#8B4513] italic">
				*Exact venue details will be confirmed and shared with registered participants soon!
			</p>
		</div>
		
		<div class="mt-12">
			<p class="text-xl font-serif text-[#8B4513] mb-6">
				Sign up today for an amazing adventure. Spots are limited!
			</p>
			<a
				href="https://forms.hackclub.com/daydream-Toronto"
				target="_blank"
				class="inline-block px-8 py-4 bg-pink border-b-4 border-b-pink-dark text-white rounded-full hover:shadow-[0_4px_0_0_theme(colors.pink.dark)] hover:-translate-y-[4px] active:border-transparent active:shadow-none active:translate-y-0 transition-all duration-100 font-sans text-xl cursor-pointer relative overflow-hidden"
			>
				Sign up for Daydream Toronto
				<img
					src="button-clouds.svg" 
					alt="" 
					class="absolute bottom-0 left-1/2 -translate-x-1/2 w-auto object-contain pointer-events-none"
				>
			</a>
		</div>
	</div>
</div>

<div class="w-full h-16 bg-[#FCEFC5]"></div>

<div class="flex flex-row flex-wrap w-full h-auto bg-gradient-to-b from-[#FCEFC5] to-[#FEC1CF] px-36 max-md:px-8 pb-50 max-sm:pb-24 relative" id="islands-container">

	<img src="/clouds-left-2.png" alt="" class="absolute left-0 w-3/12 top-12">
	<img src="/clouds-left-3.png" alt="" class="absolute left-0 w-2/12 bottom-32">
	<img src="/clouds-right-2.png" alt="" class="absolute right-0 w-3/12 bottom-0">

	<!-- SVG Path Overlay -->
	<svg class="absolute inset-0 w-full h-full pointer-events-none z-0" id="path-svg">
		<path id="dotted-path" stroke="rgba(255,255,255,0.5)" stroke-width="3" fill="none" stroke-dasharray="8,8" opacity="0.7"></path>
	</svg>

	<img src="paper-airplane.png" alt="Paper airplane" class="h-16 absolute z-10" id="paper-airplane">

	<div class="flex flex-col items-center w-max basis-1/2 max-md:basis-full max-md:w-full z-10">
		<div class="relative translate-y-8 max-md:translate-y-4">
			<img src="/letter-1-front.png" alt="" class="object-contain absolute -bottom-16 -left-13 w-28 h-28">
			<img src="/letter-1-back.png" alt="" class="object-contain absolute -bottom-16 -left-13 w-28 h-28 -z-10">
			<div class="bg-[#FFF5FA] border-2 border-[#AA8B83] rounded-2xl text-xl font-serif p-6 w-56 text-center max-md:w-64 max-md:text-lg" data-point="1">
				<span class="font-sans text-[#E472AB] font-bold text-[1.3rem] mr-1">#1:</span> Check in & meet your team
			</div>
		</div>
		<img src="/island-1.png" alt="" class="w-72 h-72 object-contain max-md:w-64 max-md:h-64">
	</div>

	<div class="flex flex-col items-center w-max basis-1/2 max-md:basis-full max-md:w-full translate-y-24 max-md:translate-y-8 z-10">
		<div class="relative translate-y-24 max-md:translate-y-4">
			<img src="/letter-2-front.png" alt="" class="object-contain absolute -bottom-16 -right-13 w-28 h-28">
			<img src="/letter-2-back.png" alt="" class="object-contain absolute -bottom-16 -right-13 w-28 h-28 -z-10">
			<div class="bg-[#FFF5FA] border-2 border-[#AA8B83] rounded-2xl text-xl font-serif p-6 w-56 text-center max-md:w-64 max-md:text-lg" data-point="2">
				<span class="font-sans text-[#639DEB] font-bold text-[1.3rem] mr-1">#2:</span> Attend workshops & learn new skills
			</div>
		</div>
		<img src="/island-3.png" alt="" class="w-86 h-86 object-contain max-md:w-64 max-md:h-64">
	</div>
	<div class="flex flex-col items-center w-max basis-1/2 max-md:basis-full max-md:w-full -translate-x-12 max-md:translate-x-0 max-md:translate-y-8 z-10">
		<div class="relative translate-y-8 max-md:translate-y-4">
			<img src="/letter-3-front.png" alt="" class="object-contain absolute -bottom-18 left-24 w-28 h-28">
			<img src="/letter-3-back.png" alt="" class="object-contain absolute -bottom-18 left-24 w-28 h-28 -z-10">
			<div class="bg-[#FFF5FA] border-2 border-[#AA8B83] rounded-2xl text-xl font-serif p-6 w-56 text-center max-md:w-64 max-md:text-lg" data-point="3">
				<span class="font-sans text-[#AB68E2] font-bold text-[1.3rem] mr-1">#3:</span> Code your game & get mentorship
			</div>
		</div>
		<img src="/island-2.png" alt="" class="w-72 h-72 object-contain max-md:w-64 max-md:h-64">
	</div>
		<div class="flex flex-col items-center w-max basis-1/2 max-md:basis-full max-md:w-full translate-y-30 max-md:translate-y-8 z-10">
		<div class="relative translate-y-24 max-md:translate-y-4">
			<img src="/letter-4-front.png" alt="" class="object-contain absolute -bottom-16 -right-13 w-28 h-28">
			<img src="/letter-4-back.png" alt="" class="object-contain absolute -bottom-16 -right-13 w-28 h-28 -z-10">
			<div class="bg-[#FFF5FA] border-2 border-[#AA8B83] rounded-2xl text-xl font-serif p-6 w-56 text-center max-md:w-64 max-md:text-lg" data-point="4">
				<span class="font-sans text-[#F2993E] font-bold text-[1.3rem] mr-1">#4:</span> Demo your project & win prizes
			</div>
		</div>
		<img src="/island-4.png" alt="" class="w-88 h-88 object-contain max-md:w-64 max-md:h-64">
	</div>

	<!-- Final Card -->
	<div class="flex flex-col items-center w-full basis-full translate-y-40 max-md:translate-y-12 z-20">
		<div class="relative">
			<div class="bg-[url('/card-final.png')] bg-contain bg-no-repeat bg-center text-2xl font-serif pt-24 px-8 w-128 h-96 text-center max-md:w-80 max-md:h-80 max-md:text-xl max-md:pt-16" data-point="5">
				<span class="font-sans text-[#F2CC32] font-bold text-[1.5rem] mr-1">#5:</span> Connect with amazing hackers!
			</div>
		</div>
	</div>

	<img src="/clouds-pink-left.png" alt="" class="absolute left-0 w-5/12 bottom-0 translate-y-32 z-50 pointer-events-none">
	<img src="/clouds-pink-right.png" alt="" class="absolute right-0 w-5/12 bottom-0 translate-y-32 z-50 pointer-events-none">

	<div class="absolute top-0 left-0 w-full h-full bg-[url('brushstroking.png')] bg-size-[100vw_100vh] bg-repeat mix-blend-overlay opacity-60 pointer-events-none bg-position-[0_100vh]"></div>
</div>

<div class="w-full bg-gradient-to-b from-[#FDC5D1] to-[#FAE3C9] items-center justify-center px-32 relative pt-36">
	<div class="max-md:absolute max-md:left-1/2 max-md:-translate-x-1/2 max-md:z-100 max-sm:pt-16">
		<div class="relative w-full max-w-3xl mx-auto min-w-72 max-md:mx-0">
			<img src="banner.png" alt="100 Cities Worldwide" class="absolute top-0 left-1/2 -translate-x-1/2 max-md:-translate-y-1/2 max-sm:translate-y-[calc(-50%-4rem)] h-48 w-auto z-100 scale-150 saturate-70 brightness-110 object-contain px-4 pointer-events-none">
			<img src="hole.png" alt="" class="w-full h-full max-w-3xl max-sm:scale-200 pointer-events-none">
			<iframe 
				src="https://felt.com/embed/map/Daydream-Events-pPFQnT34SOq6tYlb2S1IdC?loc=0%2C-73.3%2C1.7z&legend=0&cooperativeGestures=1&link=0&geolocation=0&zoomControls=1&scaleBar=0" 
				class="absolute top-0 left-0 w-full h-full border-0 max-sm:scale-200"
				style="mask: url('hole.png') no-repeat center; -webkit-mask: url('hole.png') no-repeat center; mask-size: contain; -webkit-mask-size: contain;"
				title="Felt Map"
				referrerpolicy="strict-origin-when-cross-origin">
			</iframe>
			<p class="absolute left-1/2 -translate-x-1/2 font-sans text-center text-2xl pt-12 max-sm:pt-40 max-sm:text-xl w-max max-w-[80vh] max-md:max-w-full md:px-12 text-[#60574b] z-10000 ">Join hackers from across Toronto and beyond! <br> <span class="font-bold"><a class="underline hover:text-pink" href="https://forms.hackclub.com/daydream-Toronto">Register</a> for Daydream Toronto now!</span></p>
		</div>
	</div>
	<div class="max-md:h-136"></div>
	<div class="absolute top-0 left-0 w-full h-full bg-[url('brushstroking.png')] bg-size-[100vw_100vh] bg-repeat mix-blend-overlay opacity-60 pointer-events-none"></div>
</div>

<!-- Sponsors Section -->
<div class="w-full bg-gradient-to-b from-[#FAE3C9] to-[#F5E6D3] py-8 pt-40 relative">
	<div class="absolute top-0 left-0 w-full h-full bg-[url('brushstroking.png')] bg-size-[100vw_100vh] bg-repeat mix-blend-overlay opacity-30 pointer-events-none"></div>
	
	<!-- Top Clouds -->
	<img src="/clouds-left-2.png" alt="" class="absolute left-0 w-2/12 top-0 opacity-60">
	<img src="/clouds-right-2.png" alt="" class="absolute right-0 w-2/12 top-0 opacity-60">
	
	<div class="max-w-6xl mx-auto px-8 text-center relative z-10">
		<h2 class="text-5xl font-serif italic text-[#8B4513] mb-4 relative">
			Our Amazing Sponsors
			<img src="/underline.svg" alt="" class="absolute left-1/2 -translate-x-1/2 -bottom-3 w-80 h-auto opacity-70">
		</h2>
		<p class="text-xl text-[#8B4513] mb-16 font-serif opacity-90">
			Thank you to our incredible sponsors who make Daydream Toronto possible!
		</p>
		
		<!-- Platinum Tier -->
		<div class="mb-20">
			<div class="relative inline-block mb-12">
				<h3 class="text-3xl font-serif font-bold text-[#B8860B] bg-gradient-to-b from-[#DAA520] to-[#B8860B] bg-clip-text text-transparent">
					Platinum Sponsors
				</h3>
				<div class="absolute -top-2 -left-8 -right-8 -bottom-2 bg-gradient-to-r from-[#DAA520] to-[#B8860B] rounded-xl opacity-20"></div>
			</div>
			<div class="flex justify-center items-center gap-12 flex-wrap">
				<div class="bg-[#FFF8DC] rounded-2xl p-8 shadow-lg border-4 border-[#D2B48C] transform hover:scale-105 transition-all duration-300 relative overflow-hidden">
					<div class="w-80 h-32 bg-gradient-to-r from-[#D2B48C] to-[#DEB887] rounded-lg flex items-center justify-center text-[#8B4513] font-bold text-2xl">
						Platinum Sponsor Logo
					</div>
					<img src="button-clouds.svg" alt="" class="absolute bottom-0 left-1/2 -translate-x-1/2 w-full opacity-30 pointer-events-none">
				</div>
			</div>
		</div>
		
		<!-- Gold Tier -->
		<div class="mb-16">
			<div class="relative inline-block mb-8">
				<h3 class="text-2xl font-serif font-bold text-[#CD853F] bg-gradient-to-b from-[#DEB887] to-[#CD853F] bg-clip-text text-transparent">
					Gold Sponsors
				</h3>
				<div class="absolute -top-1 -left-6 -right-6 -bottom-1 bg-gradient-to-r from-[#DEB887] to-[#CD853F] rounded-lg opacity-15"></div>
			</div>
			<div class="flex justify-center items-center gap-8 flex-wrap">
				<div class="bg-[#FFF8DC] rounded-xl p-6 shadow-md border-3 border-[#DEB887] transform hover:scale-105 transition-all duration-300 relative overflow-hidden">
					<div class="w-48 h-20 bg-gradient-to-r from-[#DEB887] to-[#D2B48C] rounded-lg flex items-center justify-center text-[#8B4513] font-bold text-lg">
						Gold Sponsor 1
					</div>
					<img src="button-clouds.svg" alt="" class="absolute bottom-0 left-1/2 -translate-x-1/2 w-full opacity-20 pointer-events-none">
				</div>
				<div class="bg-[#FFF8DC] rounded-xl p-6 shadow-md border-3 border-[#DEB887] transform hover:scale-105 transition-all duration-300 relative overflow-hidden">
					<div class="w-48 h-20 bg-gradient-to-r from-[#DEB887] to-[#D2B48C] rounded-lg flex items-center justify-center text-[#8B4513] font-bold text-lg">
						Gold Sponsor 2
					</div>
					<img src="button-clouds.svg" alt="" class="absolute bottom-0 left-1/2 -translate-x-1/2 w-full opacity-20 pointer-events-none">
				</div>
			</div>
		</div>
		
		<!-- Silver Tier -->
		<div class="mb-12">
			<div class="relative inline-block mb-6">
				<h3 class="text-xl font-serif font-bold text-[#AA8B83] bg-gradient-to-b from-[#D2B48C] to-[#AA8B83] bg-clip-text text-transparent">
					Silver Sponsors
				</h3>
				<div class="absolute -top-1 -left-4 -right-4 -bottom-1 bg-gradient-to-r from-[#D2B48C] to-[#AA8B83] rounded-md opacity-15"></div>
			</div>
			<div class="flex justify-center items-center gap-6 flex-wrap">
				<div class="bg-[#FFF8DC] rounded-lg p-4 shadow-sm border-2 border-[#D2B48C] transform hover:scale-105 transition-all duration-300 relative overflow-hidden">
					<div class="w-32 h-16 bg-gradient-to-r from-[#D2B48C] to-[#AA8B83] rounded flex items-center justify-center text-[#8B4513] font-bold text-sm">
						Silver 1
					</div>
					<img src="button-clouds.svg" alt="" class="absolute bottom-0 left-1/2 -translate-x-1/2 w-full opacity-15 pointer-events-none">
				</div>
				<div class="bg-[#FFF8DC] rounded-lg p-4 shadow-sm border-2 border-[#D2B48C] transform hover:scale-105 transition-all duration-300 relative overflow-hidden">
					<div class="w-32 h-16 bg-gradient-to-r from-[#D2B48C] to-[#AA8B83] rounded flex items-center justify-center text-[#8B4513] font-bold text-sm">
						Silver 2
					</div>
					<img src="button-clouds.svg" alt="" class="absolute bottom-0 left-1/2 -translate-x-1/2 w-full opacity-15 pointer-events-none">
				</div>
				<div class="bg-[#FFF8DC] rounded-lg p-4 shadow-sm border-2 border-[#D2B48C] transform hover:scale-105 transition-all duration-300 relative overflow-hidden">
					<div class="w-32 h-16 bg-gradient-to-r from-[#D2B48C] to-[#AA8B83] rounded flex items-center justify-center text-[#8B4513] font-bold text-sm">
						Silver 3
					</div>
					<img src="button-clouds.svg" alt="" class="absolute bottom-0 left-1/2 -translate-x-1/2 w-full opacity-15 pointer-events-none">
				</div>
			</div>
		</div>
		
		<!-- Community Partners -->
		<div class="mb-8">
			<div class="relative inline-block mb-4">
				<h3 class="text-lg font-serif font-bold text-[#8B4513] opacity-80">
					Community Partners
				</h3>
			</div>
			<div class="flex justify-center items-center gap-4 flex-wrap">
				<div class="bg-[#FFF8DC] rounded p-3 shadow-sm border border-[#AA8B83] transform hover:scale-105 transition-all duration-300">
					<div class="w-24 h-12 bg-gradient-to-r from-[#AA8B83] to-[#8B4513] rounded flex items-center justify-center text-white font-bold text-xs">
						Partner 1
					</div>
				</div>
				<div class="bg-[#FFF8DC] rounded p-3 shadow-sm border border-[#AA8B83] transform hover:scale-105 transition-all duration-300">
					<div class="w-24 h-12 bg-gradient-to-r from-[#AA8B83] to-[#8B4513] rounded flex items-center justify-center text-white font-bold text-xs">
						Partner 2
					</div>
				</div>
				<div class="bg-[#FFF8DC] rounded p-3 shadow-sm border border-[#AA8B83] transform hover:scale-105 transition-all duration-300">
					<div class="w-24 h-12 bg-gradient-to-r from-[#AA8B83] to-[#8B4513] rounded flex items-center justify-center text-white font-bold text-xs">
						Partner 3
					</div>
				</div>
				<div class="bg-[#FFF8DC] rounded p-3 shadow-sm border border-[#AA8B83] transform hover:scale-105 transition-all duration-300">
					<div class="w-24 h-12 bg-gradient-to-r from-[#AA8B83] to-[#8B4513] rounded flex items-center justify-center text-white font-bold text-xs">
						Partner 4
					</div>
				</div>
			</div>
		</div>
		
		<!-- Sponsor CTA -->
		<div class="mt-16 p-8 bg-[#FFF8DC] bg-opacity-90 rounded-2xl border-2 border-[#AA8B83] shadow-lg relative overflow-hidden">
			<img src="button-clouds.svg" alt="" class="absolute bottom-0 left-1/2 -translate-x-1/2 w-full opacity-20 pointer-events-none">
			<h3 class="text-2xl font-serif font-bold text-[#8B4513] mb-4">
				Want to sponsor Daydream Toronto?
			</h3>
			<p class="text-[#8B4513] mb-6 font-serif">
				Help us create an amazing experience for young developers in Toronto! 
				Your support makes this event possible and helps nurture the next generation of innovators.
			</p>
			<a
				href="mailto:Toronto@daydream.hackclub.com?subject=Sponsorship%20Inquiry%20-%20Daydream%20Toronto"
				class="inline-block px-6 py-3 bg-pink border-b-4 border-b-pink-dark text-white rounded-full hover:shadow-[0_4px_0_0_theme(colors.pink.dark)] hover:-translate-y-[4px] active:border-transparent active:shadow-none active:translate-y-0 transition-all duration-100 font-sans text-lg cursor-pointer relative overflow-hidden"
			>
				Become a Sponsor
				<img
					src="button-clouds.svg" 
					alt="" 
					class="absolute bottom-0 left-1/2 -translate-x-1/2 w-auto object-contain pointer-events-none"
				>
			</a>
		</div>
	</div>
	
	<!-- Bottom Clouds -->
	<img src="/clouds-left-3.png" alt="" class="absolute left-0 w-3/12 bottom-0 opacity-50 pointer-events-none">
	<img src="/clouds-right-2.png" alt="" class="absolute right-0 w-3/12 bottom-0 opacity-50 pointer-events-none">
</div>

<div class="w-full pb-24 pt-8 max-md:pt-8 bg-gradient-to-b from-[#F5E6D3] to-[#e99cce] relative flex flex-col items-center justify-center">
	<img src="faq-clouds.png" alt="" class="w-full">
	<img src="faq.png" alt="FAQ" class="mb-12 h-24 scale-175 max-md:scale-120">

	<!-- FAQ Grid -->
	<div class="grid grid-cols-2 gap-8 max-w-6xl px-8 z-10 max-[900px]:grid-cols-1 max-md:gap-16">
		<!-- FAQ Item 1 -->
		<div class="relative transform -rotate-2">
			<img src="window-3.png" alt="window" class="w-full h-full object-contain max-md:scale-130 max-xl:scale-110 max-lg:scale-115">
			<div class="absolute top-20 left-12 right-12 bottom-16 flex flex-col items-center justify-center text-center px-24 opacity-70 max-[900px]:mx-[15vw] max-sm:mx-0 max-sm:px-5 max-lg:px-14 max-xl:px-18">
				<h3 class="text-xl font-serif font-bold mb-4 max-lg:mb-0 max-md:text-base">Who can participate in Daydream Toronto?</h3>
				<p class="text-sm">All high-school & upper-middle-school aged students from Toronto and the surrounding region are welcome!</p>
		</div>
		</div>

		<!-- FAQ Item 2 -->
		<div class="relative transform rotate-1">
			<img src="window-4.png" alt="window" class="w-full h-full object-contain max-md:scale-130 max-xl:scale-110 max-lg:scale-115">
			<div class="absolute top-20 left-12 right-12 bottom-16 flex flex-col items-center justify-center text-center px-24 opacity-70 max-[900px]:mx-[15vw] max-sm:mx-0 max-sm:px-5 max-lg:px-14 max-xl:px-18">
				<h3 class="text-xl font-serif font-bold mb-4 max-lg:mb-0 max-md:text-base">Where is Daydream Toronto happening?</h3>
				<p class="text-sm">The event will be held in Toronto - exact location details will be shared with registered participants!</p>
			</div>
		</div>

		<!-- FAQ Item 3 -->
		<div class="relative transform rotate-2">
			<img src="window-2.png" alt="window" class="w-full h-full object-contain max-md:scale-130 max-xl:scale-110 max-lg:scale-115">
			<div class="absolute top-20 left-12 right-12 bottom-16 flex flex-col items-center justify-center text-center px-24  opacity-70 max-[900px]:mx-[15vw] max-sm:mx-0 max-sm:px-5 max-lg:px-14 max-xl:px-18">
				<h3 class="text-xl font-serif font-bold mb-4 max-lg:mb-0 max-md:text-base">All this, for free?</h3>
				<p class="text-sm">Yep! Food, swag, workshops, and good vibes are all included. Plus travel support for those coming from outside Toronto!</p>
			</div>
		</div>

		<!-- FAQ Item 4 -->
		<div class="relative transform -rotate-1">
			<img src="window-1.png" alt="window" class="w-full h-full object-contain max-md:scale-130 max-xl:scale-110 max-lg:scale-115">
			<div class="absolute top-20 left-12 right-12 bottom-16 flex flex-col items-center justify-center text-center px-24  opacity-70 max-[900px]:mx-[15vw] max-sm:mx-0 max-sm:px-5 max-lg:px-14 max-xl:px-18">
				<h3 class="text-xl font-serif font-bold mb-4 max-lg:mb-0 max-md:text-base">What do I need to bring?</h3>
				<p class="text-sm">Your laptop, chargers, toiletries, any photo ID, a refillable water bottle, and an open mind! Since it's 24 hours, bring a sleeping bag, pillow, and deodorant too.</p>
			</div>
		</div>

		<!-- FAQ Item 5 -->
		<div class="relative transform rotate-1">
			<img src="window-4.png" alt="window" class="w-full h-full object-contain max-md:scale-130 max-xl:scale-110 max-lg:scale-115">
			<div class="absolute top-20 left-12 right-12 bottom-16 flex flex-col items-center justify-center text-center px-24 opacity-70 max-[900px]:mx-[15vw] max-sm:mx-0 max-sm:px-5 max-lg:px-14 max-xl:px-18">
				<h3 class="text-xl font-serif font-bold mb-4 max-lg:mb-1 max-md:text-base">How do I register for Daydream Toronto?</h3>
				<p class="text-sm">Simply enter your email above or click the registration link! We'll send you all the details about location, schedule, and what to bring.</p>
			</div>
		</div>

		<!-- FAQ Item 6 -->
		<div class="relative transform rotate-1">
			<img src="window-3.png" alt="window" class="w-full h-full object-contain max-md:scale-130 max-xl:scale-110 max-lg:scale-115">
			<div class="absolute top-20 left-12 right-12 bottom-16 flex flex-col items-center justify-center text-center px-24 opacity-70 max-[900px]:mx-[15vw] max-sm:mx-0 max-sm:px-5 max-lg:px-14 max-xl:px-18">
				<h3 class="text-xl font-serif font-bold mb-4 max-lg:mb-0 max-md:text-base">I'm not good at coding. Can I still participate?</h3>
				<p class="text-sm">This game jam is for all skill levels! We'll have workshops and other events so join us and let's learn together.</p>
			</div>
		</div>

		<!-- FAQ Item 7 -->
		<div class="relative transform -rotate-2">
			<img src="window-2.png" alt="window" class="w-full h-full object-contain max-md:scale-130 max-xl:scale-110 max-lg:scale-115">
			<div class="absolute top-20 left-12 right-12 bottom-16 flex flex-col items-center justify-center text-center px-24 opacity-70 max-[900px]:mx-[15vw] max-sm:mx-0 max-sm:px-5 max-lg:px-14 max-xl:px-18">
				<h3 class="text-xl font-serif font-bold mb-4 max-lg:mb-0 max-md:text-base">What if I have more questions?</h3>
				<p class="text-sm">Contact us! Feel free to reach out in the #daydream-Toronto channel on Hack Club Slack or email us at Toronto@daydream.hackclub.com</p>
			</div>
		</div>

		<!-- FAQ Item 8 -->
		<div class="relative transform -rotate-1">
			<img src="window-1.png" alt="window" class="w-full h-full object-contain max-md:scale-130 max-xl:scale-110 max-lg:scale-115">
			<div class="absolute top-20 left-12 right-12 bottom-16 flex flex-col items-center justify-center text-center px-24 opacity-70 max-[900px]:mx-[15vw] max-sm:mx-0 max-sm:px-5 max-lg:px-14 max-xl:px-18">
				<h3 class="text-xl font-serif font-bold mb-4 max-lg:mb-0 max-md:text-base">What can I make at Daydream Toronto?</h3>
				<p class="text-sm">ANY type of game based on the theme! Platformer, visual novel, clicker game, VR experience, mobile game - be as creative as possible!</p>
			</div>
		</div>
	</div>

	<div class="absolute top-0 left-0 w-full h-full bg-[url('brushstroking.png')] bg-size-[100vw_100vh] bg-repeat mix-blend-overlay opacity-60 pointer-events-none"></div>
</div>

<div class="w-full bg-[#FFFFF8] relative min-h-80">
	<div class="absolute top-0 left-0 w-full h-full bg-[url('/noise.png')] bg-repeat opacity-10 pointer-events-none z-0"></div>
	<div class="opacity-60 absolute w-full h-32 bg-[url('brushstroking.png')] bg-repeat-x z-10 bg-size-[100vw_100vh] mix-blend-overlay" style="mask-image: url(/footer-clouds.png); mask-size: contain; mask-repeat: repeat-x; -webkit-mask-image: url(/footer-clouds.png); -webkit-mask-size: contain; -webkit-mask-repeat: repeat-x;"></div>
	<div class="w-full h-32 bg-[#e99cce] z-5" style="mask-image: url(/footer-clouds.png); mask-size: contain; mask-repeat: repeat-x; -webkit-mask-image: url(/footer-clouds.png); -webkit-mask-size: contain; -webkit-mask-repeat: repeat-x;"></div>

	<!-- Footer Text -->
	<div class="absolute bottom-20 left-32 text-center z-20 max-md:bottom-12 max-md:left-8 max-md:right-4 max-md:text-left">
		<p class="text-gray-700 mb-2">Made with ♡ by teenagers, for teenagers at Hack Club</p>
		<div class="flex space-x-4 max-md:flex-col max-md:space-x-0 max-md:space-y-2">
			<a href="https://hackclub.com" class="underline text-gray-700 hover:text-gray-900 transition-colors ">Hack Club</a>
			<span class="text-gray-700 max-md:hidden">・</span>
			<a href="https://hackclub.com/slack" class="underline text-gray-700 hover:text-gray-900 transition-colors ">Slack</a>
			<span class="text-gray-700 max-md:hidden">・</span>
			<a href="https://hackclub.com/clubs" class="underline text-gray-700 hover:text-gray-900 transition-colors ">Clubs</a>
			<span class="text-gray-700 max-md:hidden">・</span>
			<a href="https://hackclub.com/hackathons" class="underline text-gray-700 hover:text-gray-900 transition-colors ">Hackathons</a>
		</div>
	</div>

	<div class="max-sm:hidden absolute bottom-2 right-16 h-2/3 aspect-square bg-[url('brushstroking.png')] bg-repeat z-10 bg-size-[100vw_100vh] mix-blend-overlay" style="mask-image: url(/thought-bubbles.png); mask-size: contain; mask-repeat: no-repeat; -webkit-mask-image: url(/thought-bubbles.png); -webkit-mask-size: contain; -webkit-mask-repeat: no-repeat;"></div>
	<div class="max-sm:hidden absolute bottom-2 right-16 h-2/3 aspect-square bg-[#e99cce]" style="mask-image: url(/thought-bubbles.png); mask-size: contain; mask-repeat: no-repeat; -webkit-mask-image: url(/thought-bubbles.png); -webkit-mask-size: contain; -webkit-mask-repeat: no-repeat;"></div>
</div>

<!-- hidden form for newsletter signup -->
<form
	id="hidden-signup-form"
	method="post"
	action="https://app.loops.so/api/newsletter-form/clo3frr4v02f3jv0qqu6hgfqs"
	target="hidden-iframe"
	style="display: none;"
>
	<input type="email" name="email" required>
	<input type="hidden" name="mailingLists" value="cmd3c94kz0hvz0iwt7ps28cyd">
	<button type="submit">Sign up</button>
</form>

<!-- hidden iframe to receive form submission -->
<!-- svelte-ignore a11y_missing_attribute -->
<iframe name="hidden-iframe" style="display: none;"></iframe>
