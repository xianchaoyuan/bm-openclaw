// 编译control-ui
pnpm --filter openclaw-control-ui build

// 运行gateway服务
pnpm openclaw gateway --dev --auth none --port 19001