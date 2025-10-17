# Cricket Counter App

A Svelte-based cricket umpire counter app optimized for iPhone 14.

## Features

- **Ball & Over Counting**: Track balls (0-5) and overs with automatic progression
- **Touch Feedback**: Haptic vibration and audio feedback on ball taps
- **Undo Functionality**: Slide-to-remove control with hold-to-confirm
- **State Persistence**: Automatically saves progress to localStorage
- **Screen Wake Lock**: Prevents screen from sleeping while app is active
- **Reset Options**: Reset current over or entire game with confirmation
- **iPhone 14 Optimized**: Designed for 390x844px viewport

## Development

```bash
# Install dependencies
npm install

# Start development server
npm run dev

# Build for production
npm run build

# Preview production build
npm run preview
```

## Deployment

The app is configured for static deployment to S3 with CloudFront:

1. Run `npm run build` to create the production build
2. Upload the contents of the `build` directory to your S3 bucket
3. Configure CloudFront to serve the static files
4. Set up proper caching headers for optimal performance

## Usage

- **Tap the cricket ball** to increment the ball count
- **Slide the "REMOVE" control** from right to left and hold for 2 seconds to undo a ball
- **Tap "RESET"** to choose between resetting the current over or the entire game
- The app automatically saves your progress and prevents the screen from sleeping

## Browser Support

- Modern mobile browsers (iOS Safari, Chrome Mobile)
- Requires support for:
  - Touch events
  - Web Audio API
  - Vibration API
  - Wake Lock API
  - localStorage
