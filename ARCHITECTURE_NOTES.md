# Architecture Notes

## File Organization

### Source of Truth
All agent definitions are stored in `.claude/agents/` and skills in `.claude/skills/`. These are the authoritative versions.

### Plugin Structure
Plugin directories (`flutter-all/`, `flutter-ui/`, etc.) contain:
- **Symlinks** to agent files (not duplicates)
- `README.md` describing the plugin
- `.claude-plugin/plugin.json` with metadata

### Why Symlinks?
Using symlinks instead of file duplication provides several benefits:
1. **Single source of truth**: Edit once in `.claude/agents/`, changes reflect everywhere
2. **Reduced storage**: No file duplication
3. **Easier maintenance**: Updates propagate automatically
4. **Git-friendly**: Symlinks are tracked in git

### Platform Compatibility
- **macOS/Linux**: Symlinks work natively
- **Windows**: May require administrator privileges or Git config `core.symlinks=true`

## Documentation Structure

### User-Facing Documentation
- `README.md` - Main project overview and installation
- `PLUGIN_INSTALLATION.md` - Detailed installation guide
- `AGENT_USAGE_SCENARIOS.md` - Practical examples and use cases

### Developer/Contributor Documentation
- `CONTRIBUTING.md` - Consolidated architecture, implementation, and contribution guide
- `FLUTTER_AGENT_ARCHITECTURE_PLAN.md` - Detailed architecture specification (reference)
- `IMPLEMENTATION_GUIDE.md` - Implementation details (reference)
- `WORKFLOW_DIAGRAMS.md` - Visual workflow diagrams (reference)

### Documentation Consolidation
`CONTRIBUTING.md` serves as the primary contributor guide, with references to the detailed docs above for deep dives.

## Maintenance Guidelines

### When Updating Agents
1. Edit files in `.claude/agents/` (source of truth)
2. Changes automatically reflect in all plugins via symlinks
3. Update relevant documentation
4. Test changes with actual Flutter projects

### When Adding New Agents
1. Create agent file in `.claude/agents/`
2. Add symlink in relevant plugin(s)
3. Update `marketplace.json` if creating new plugin
4. Add usage examples to `AGENT_USAGE_SCENARIOS.md`
5. Update plugin README files

### When Modifying Plugins
1. Plugin structure changes: Update directory structure
2. Plugin metadata: Edit `.claude-plugin/plugin.json`
3. Plugin description: Edit plugin's `README.md`
4. Marketplace: Update `.claude-plugin/marketplace.json`

## Version Control

### What's Committed
- All source files in `.claude/`
- All symlinks in plugin directories
- All documentation
- Plugin metadata (plugin.json)
- Marketplace configuration

### What's Ignored
- macOS `.DS_Store` files
- IDE files (`.vscode/`, `.idea/`)
- Backup files (`*.bak`, `*.backup`)

## Future Considerations

### Potential Expansions
- Flutter Web specialist agents
- Flutter Desktop (macOS, Windows, Linux) agents
- Additional skills for advanced patterns
- CI/CD integration agents

### Community Contributions
- Follow CONTRIBUTING.md guidelines
- Test thoroughly before submitting
- Update documentation
- Maintain backward compatibility
